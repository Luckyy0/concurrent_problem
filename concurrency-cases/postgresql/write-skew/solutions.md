# Giải pháp guard row, locking và serializable

## Mục tiêu thiết kế

Mọi transaction có thể làm thay đổi quy tắc bất biến của danh sách trực phải gặp cùng một điểm xung đột xác thực (authoritative conflict point) hoặc được database kiểm tra như một luồng thực thi tuần tự hóa (serializable execution).

Hành vi của phía thất bại (loser) phải rõ ràng: block rồi bị từ chối, timeout, số row bị ảnh hưởng bằng `0`, hoặc bị hủy bằng mã lỗi `40001` và thử lại có giới hạn.

## Giải pháp 1 — Lock roster guard row

`on_call_roster` đã là thực thể cha ổn định; có thể dùng nó làm guard:

```java
public interface OnCallRosterRepository
    extends JpaRepository<OnCallRoster, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("""
        select r
          from OnCallRoster r
         where r.id = :rosterId
        """)
    Optional<OnCallRoster> findForInvariantChange(UUID rosterId);
}
```

PostgreSQL tương đương:

```sql
select roster_id
from on_call_roster
where roster_id = :rosterId
for update;
```

Dịch vụ (Service):

```java
@Service
public class GuardedOnCallService {
    private final OnCallRosterRepository rosters;
    private final OnCallAssignmentRepository assignments;

    public GuardedOnCallService(
        OnCallRosterRepository rosters,
        OnCallAssignmentRepository assignments
    ) {
        this.rosters = rosters;
        this.assignments = assignments;
    }

    @Transactional
    public LeaveResult leaveOnCall(UUID rosterId, UUID operatorId) {
        rosters.findForInvariantChange(rosterId)
            .orElseThrow(RosterNotFoundException::new);

        OnCallAssignment own = assignments
            .findAssignment(rosterId, operatorId)
            .orElseThrow(AssignmentNotFoundException::new);
        if (!own.isOnCall()) {
            return LeaveResult.alreadyOffCall();
        }

        long count = assignments.countOnCall(rosterId);
        if (count <= 1) {
            return LeaveResult.lastOperatorRequired();
        }

        own.leave();
        assignments.flush();
        return LeaveResult.accepted();
    }
}
```

Hành vi (Behavior):

1. A khóa dòng danh sách trực (locks roster row).
2. B nỗ lực thực hiện `FOR UPDATE` tương tự và bị chặn (blocks).
3. A đếm số lượng `2`, cập nhật Alice, commits; guard lock được giải phóng.
4. B lấy được guard, tải trạng thái và kết quả đếm hiện tại là `1`.
5. B trả về `LAST_OPERATOR_REQUIRED` mà không cập nhật Bob.

> **Nói ngắn gọn:** guard row làm cho quy tắc bất biến mức danh sách trực có một row vật lý mà mọi transaction cạnh tranh đều buộc phải đi qua.

Mọi luồng thao tác thêm/xóa/rời ca/kích hoạt lại phải thực hiện lock guard trước. Với nhiều danh sách trực, hãy thực hiện lock theo thứ tự roster ID đã sắp xếp. Giữ transaction ngắn, dùng giới hạn `lock_timeout`, không gọi dịch vụ từ xa khi đang giữ lock.

## Giải pháp 2 — Bộ đếm on-call xác thực (Authoritative on-call counter)

Thêm bộ đếm vào danh sách trực:

```sql
alter table on_call_roster
    add column on_call_count integer not null,
    add constraint ck_on_call_count_positive
        check (on_call_count >= 1);
```

Một luồng nỗ lực rời ca (leave attempt) vững chắc thực hiện cập nhật row của chính mình rồi giảm giá trị guard có điều kiện (conditional-decrement guard) trong cùng một transaction:

```java
public interface OnCallAssignmentCommands {
    @Modifying
    @Query(
        value = """
            update on_call_assignment
               set on_call = false,
                   version = version + 1
             where roster_id = :rosterId
               and operator_id = :operatorId
               and on_call = true
            """,
        nativeQuery = true
    )
    int markOffCall(UUID rosterId, UUID operatorId);
}

public interface OnCallRosterCommands {
    @Modifying
    @Query(
        value = """
            update on_call_roster
               set on_call_count = on_call_count - 1
             where roster_id = :rosterId
               and on_call_count > 1
            """,
        nativeQuery = true
    )
    int decrementIfAnotherRemains(UUID rosterId);
}
```

```java
@Transactional
public LeaveResult leaveWithCounter(UUID rosterId, UUID operatorId) {
    int changed = assignmentCommands.markOffCall(rosterId, operatorId);
    if (changed == 0) {
        return LeaveResult.alreadyOffCall();
    }

    int decremented = rosterCommands.decrementIfAnotherRemains(rosterId);
    if (decremented == 0) {
        throw new LastOperatorRequiredException(rosterId);
    }
    return LeaveResult.accepted();
}
```

`LastOperatorRequiredException` là runtime exception được ánh xạ thành kết quả trả về của hệ thống (domain response) bên ngoài transaction; bước rollback khôi phục lại cập nhật của lệnh phân công. Hai tiến trình đồng thời A và B cập nhật các phân công khác nhau rồi tranh chấp trên bộ đếm danh sách trực: bên thứ nhất giảm từ `2 -> 1`, bên thứ hai thấy điều kiện sai và tự động rollback cập nhật của chính mình.

Tất cả luồng biến đổi dữ liệu (mutation paths) phải dùng thứ tự lock thống nhất (consistent lock order) và giữ bộ đếm đồng bộ. Có đối soát:

```sql
select r.roster_id, r.on_call_count,
       count(a.*) filter (where a.on_call) as actual
from on_call_roster r
left join on_call_assignment a on a.roster_id = r.roster_id
group by r.roster_id, r.on_call_count
having r.on_call_count <> count(a.*) filter (where a.on_call);
```

## Giải pháp 3 — Lock toàn bộ tập hợp phân công liên quan

Nếu không có row cha thì lock tất cả các rows phân công hiện tại theo một thứ tự tất định:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("""
    select a
      from OnCallAssignment a
     where a.rosterId = :rosterId
     order by a.operatorId
    """)
List<OnCallAssignment> findAllForUpdate(UUID rosterId);
```

A khóa các rows của Alice/Bob; B phải chờ. Sau khi A commit, lệnh của B ở mức `READ COMMITTED` sẽ lấy được phiên bản hiện tại và buộc phải đánh giá lại tập on-call hiện tại trước khi thực hiện rời ca.

Nhược điểm: giao thức phải xử lý được thao tác INSERT phân công đồng thời. Việc lock các rows có sẵn không khóa được một row tạo mới trong tương lai; một guard cha ổn định thường rõ ràng hơn. Thứ tự lock khác nhau sẽ tạo ra deadlock; kích thước tập hợp lớn làm tăng tiêu thụ lock, memory và độ trễ.

## Giải pháp 4 — PostgreSQL `SERIALIZABLE`

Tiến hành nỗ lực (Attempt):

```java
@Service
public class SerializableLeaveAttempt {
    private final OnCallAssignmentRepository assignments;

    @Transactional(
        isolation = Isolation.SERIALIZABLE,
        propagation = Propagation.REQUIRES_NEW
    )
    public LeaveResult run(UUID rosterId, UUID operatorId) {
        long count = assignments.countOnCall(rosterId);
        OnCallAssignment own = assignments
            .findAssignment(rosterId, operatorId)
            .orElseThrow(AssignmentNotFoundException::new);

        if (!own.isOnCall()) {
            return LeaveResult.alreadyOffCall();
        }
        if (count <= 1) {
            return LeaveResult.lastOperatorRequired();
        }

        own.leave();
        assignments.flush();
        return LeaveResult.accepted();
    }
}
```

Vòng lặp thử lại có giới hạn bên ngoài:

```java
public LeaveResult leave(UUID rosterId, UUID operatorId) {
    for (int attemptNumber = 1; attemptNumber <= 3; attemptNumber++) {
        try {
            return attempt.run(rosterId, operatorId);
        } catch (CannotSerializeTransactionException ex) {
            if (attemptNumber == 3) {
                throw ex;
            }
            backoff.pauseWithJitter(attemptNumber);
        }
    }
    throw new IllegalStateException("unreachable");
}
```

Trình điều phối thử lại (retry orchestrator) và thao tác cố gắng thực thi transaction là hai Spring beans để proxy có thể tạo ra một transaction mới về mặt vật lý. Phía thất bại sẽ thực hiện rollback; lệnh thử lại sẽ tải trạng thái của bên đã chiến thắng và trả về `LAST_OPERATOR_REQUIRED`.

SSI tránh việc chặn rõ ràng (explicit blocking guard) nhưng có chi phí abort và thử lại. Danh sách trực thao tác liên tục (hot rosters) có thể gây khuếch đại thử lại; đo đạc số lần thử lại và cạn kiệt số lần thử là điều bắt buộc.

## Giải pháp 5 — Constraint trigger với guard rõ ràng

`CHECK` ở mức row hoặc unique constraint không diễn đạt được quy tắc phải có ít nhất một row anh em. Một constraint trigger chỉ an toàn nếu nó lock roster guard trước khi tiến hành bước xác thực tổng hợp:

```sql
select 1
from on_call_roster
where roster_id = :rosterId
for update;

select count(*)
from on_call_assignment
where roster_id = :rosterId
  and on_call;
```

Trigger không có giao thức lock vẫn sẽ xảy ra race condition. Logic trong database tăng độ bao phủ thực thi cho truy vấn SQL trực tiếp nhưng lại tăng độ phức tạp khi migration và gỡ lỗi; stored procedure có thể là một ranh giới (boundary) rõ ràng hơn.

## Tại sao `@Version` và tính duy nhất không đủ

SQL được tạo ra:

```sql
update on_call_assignment
set on_call=false, version=version+1
where assignment_id=:id and version=:oldVersion;
```

Khi số row bị ảnh hưởng là `0` thì phát hiện thao tác ghi lỗi thời (stale write) cùng lúc trên một phân công; write skew có số row ảnh hưởng là `1` trên cả Alice và Bob. Ràng buộc unique cũng không biểu diễn quy tắc đếm có giới hạn dưới.

## So sánh ưu nhược điểm (Trade-off comparison)

| Cách tiếp cận | Điểm xung đột | Xử lý bên thất bại | Cạnh tranh | Độ phức tạp |
| --- | --- | --- | --- | --- |
| Guard `FOR UPDATE` | Roster row | block rồi từ chối/timeout | Theo hot roster | Thấp |
| Bộ đếm có điều kiện | Roster counter | affected-row `0` + rollback | Theo hot roster | Cần bộ đếm và đối soát |
| Lock toàn bộ rows | Assignment set | block/timeout | Theo kích thước set | Cần thứ tự lock |
| `SERIALIZABLE` | Các phụ thuộc SSI | gặp `40001`, thử lại | Thử lại khi có xung đột | Vận hành vòng lặp thử lại |
| JVM lock | Process memory | chờ ở bộ nhớ cục bộ | Chỉ áp dụng cục bộ | Không hỗ trợ đa tiến trình |

## Lỗi và phục hồi (Failure and recovery)

- Tiến trình giữ guard bị rollback/crash: việc phân công sẽ rollback, giải phóng row lock.
- Không thể giảm bộ đếm: runtime exception rollback các cập nhật phân công trước đó.
- Giao dịch thua SSI: transaction không sử dụng được nữa; thử lại bằng transaction mới.
- Timeout hoặc deadlock: rollback toàn bộ nỗ lực; việc thử lại an toàn có giới hạn nếu lệnh có tính lũy đẳng (idempotent).
- Crash sau khi commit hoặc trước khi phản hồi: trạng thái có điều kiện hoặc khóa lũy đẳng (idempotency key) phát lại kết quả mà không làm giảm dữ liệu thêm một lần nữa.
- Lệnh gửi thông báo ra bên ngoài (external notification): thực hiện outbox bền bỉ chỉ khi transaction thành công.

## Khuyến nghị (Recommendation)

Với các thực thể danh sách trực cha ổn định, việc đặt khóa guard-row là thao tác cơ bản dễ tiến hành kiểm toán. Dùng bộ đếm phù hợp khi các thao tác đọc danh sách trực cần đếm nhanh và đội ngũ có quy trình đối soát dữ liệu (reconciliation).
`SERIALIZABLE` phù hợp khi quy tắc bất biến phức tạp hoặc nhiều luồng thay đổi khó quy về một điểm guard thống nhất, miễn là tính năng thử lại toàn bộ transaction được vận hành tốt.

## Danh sách kiểm tra môi trường production (Production checklist)

- [ ] Phạm vi của quy tắc bất biến phải áp dụng trên toàn danh sách trực, không nhầm lẫn với từng row phân công riêng lẻ.
- [ ] Mọi luồng thay đổi trạng thái phải dùng cùng một hợp đồng cấu trúc guard, bộ đếm hoặc SSI.
- [ ] Chế độ cô lập đã được xác minh tính hiệu quả.
- [ ] Có cách xử lý có giới hạn cho các lỗi `40001`, `40P01`, `55P03`.
- [ ] Luồng thử lại tạo một transaction mới và tải lại toàn bộ danh sách trực.
- [ ] Việc lock nhiều danh sách trực và nhân viên luôn theo một thứ tự tất định.
- [ ] Không có truy vấn I/O từ xa khi đang giữ database locks.
- [ ] Các cảnh báo về đối soát lệch giữa row và bộ đếm, cũng như về trạng thái không an toàn phải tồn tại.
- [ ] Lệnh trùng lặp không thực hiện rời ca hay giảm số đếm lần thứ hai.
- [ ] Có PostgreSQL Testcontainers để chạy hồi quy quy tắc bất biến cuối cùng của số lượng danh sách trực.
