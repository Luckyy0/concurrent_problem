# Cài đặt versioned-row bị lỗi

## Thực thể phân công (Assignment entity)

```java
@Entity
@Table(
    name = "on_call_assignment",
    uniqueConstraints = @UniqueConstraint(
        name = "uk_roster_operator",
        columnNames = {"roster_id", "operator_id"}
    )
)
public class OnCallAssignment {
    @Id
    @Column(name = "assignment_id", nullable = false)
    private UUID id;

    @Column(name = "roster_id", nullable = false)
    private UUID rosterId;

    @Column(name = "operator_id", nullable = false)
    private UUID operatorId;

    @Column(name = "on_call", nullable = false)
    private boolean onCall;

    @Version
    @Column(nullable = false)
    private long version;

    protected OnCallAssignment() {
    }

    public void leave() {
        onCall = false;
    }

    public UUID getRosterId() {
        return rosterId;
    }

    public boolean isOnCall() {
        return onCall;
    }
}
```

`@Version` bảo vệ việc ghi dữ liệu cũ trên cùng một row phân công. Nó không tạo một phiên bản chia sẻ cho toàn danh sách trực.

## Repository

```java
public interface OnCallAssignmentRepository
    extends JpaRepository<OnCallAssignment, UUID> {

    @Query("""
        select count(a)
          from OnCallAssignment a
         where a.rosterId = :rosterId
           and a.onCall = true
        """)
    long countOnCall(UUID rosterId);

    @Query("""
        select a
          from OnCallAssignment a
         where a.rosterId = :rosterId
           and a.operatorId = :operatorId
        """)
    Optional<OnCallAssignment> findAssignment(
        UUID rosterId,
        UUID operatorId
    );
}
```

## Dịch vụ lỗi (Broken service)

```java
@Service
public class OnCallService {
    private final OnCallAssignmentRepository assignments;

    public OnCallService(OnCallAssignmentRepository assignments) {
        this.assignments = assignments;
    }

    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public LeaveResult leaveOnCall(UUID rosterId, UUID operatorId) {
        long onCallCount = assignments.countOnCall(rosterId);
        if (onCallCount <= 1) {
            return LeaveResult.lastOperatorRequired();
        }

        OnCallAssignment own = assignments
            .findAssignment(rosterId, operatorId)
            .orElseThrow(AssignmentNotFoundException::new);

        if (!own.isOnCall()) {
            return LeaveResult.alreadyOffCall();
        }

        own.leave();
        return LeaveResult.accepted();
    }
}
```

`REPEATABLE_READ` thường được thêm với ý định “giữ kết quả đếm ổn định”. Điều đó đúng cho các lần đọc lặp lại trong từng transaction, nhưng chính stable snapshots cho phép A và B cùng tiếp tục tin rằng số lượng là `2`.

> **Nói ngắn gọn:** cả hai thao tác UPDATE của Hibernate đều có điều kiện phiên bản và có số row bị ảnh hưởng là `1`; optimistic locking không báo lỗi vì Alice và Bob là hai rows khác nhau.

## Câu lệnh SQL khi Hibernate flush

A:

```sql
update on_call_assignment
set on_call = false,
    version = 1
where assignment_id = :aliceAssignmentId
  and version = 0;
-- affected rows = 1
```

B:

```sql
update on_call_assignment
set on_call = false,
    version = 1
where assignment_id = :bobAssignmentId
  and version = 0;
-- affected rows = 1
```

Hibernate chỉ ném `OptimisticLockException` (hoặc Spring optimistic-lock exception) khi thao tác cập nhật có version có số row bị ảnh hưởng là `0`. Ở đây cả hai là `1`, nên không có tín hiệu xung đột để thử lại.

## Trình tự xen kẽ cụ thể (Concrete interleaving)

```text
A BEGIN REPEATABLE READ
B BEGIN REPEATABLE READ
A COUNT on_call=true -> 2
B COUNT on_call=true -> 2
A load Alice version=0
B load Bob version=0
A UPDATE Alice false, version=1 -> affected 1
B UPDATE Bob false, version=1   -> affected 1
A COMMIT
B COMMIT
final COUNT on_call=true -> 0
```

Row locks:

- Thao tác UPDATE của A locks Alice row tới lúc commit;
- Thao tác UPDATE của B locks Bob row tới lúc commit;
- Các locks không xung đột;
- Lệnh đếm (COUNT) thông thường không lock điều kiện danh sách trực.

## Schema tương đương

```sql
create table on_call_roster (
    roster_id uuid primary key,
    name varchar(100) not null,
    active boolean not null
);

create table on_call_assignment (
    assignment_id uuid primary key,
    roster_id uuid not null references on_call_roster(roster_id),
    operator_id uuid not null,
    on_call boolean not null,
    version bigint not null,
    unique (roster_id, operator_id)
);
```

Ràng buộc unique ngăn việc gán nhân viên trùng lặp; nó không ép buộc “ít nhất một row on_call=true”.

## Điều kiện tái hiện (Preconditions)

1. Danh sách trực ban đầu có đúng hai nhân viên on-call.
2. A và B có physical transactions và connections độc lập.
3. Cả hai thực hiện đếm trước khi bất kỳ ai commit thao tác UPDATE.
4. A cập nhật dòng của Alice, B cập nhật dòng của Bob.
5. Mức cô lập hiệu lực là `READ COMMITTED` hoặc PostgreSQL `REPEATABLE READ`.
6. Không có guard-row lock, bộ đếm hoặc thử lại tuần tự hóa (serializable retry).
7. Test không có transaction bên ngoài (outer transaction) che lấp việc commit.

## Những cách sửa chưa đủ

### Chỉ thêm `@Version`

Đã có; phạm vi phiên bản tính theo dòng phân công, trong khi phạm vi quy tắc tính theo danh sách trực.

### Nâng từ `READ COMMITTED` lên `REPEATABLE READ`

Ngăn chặn việc đọc không lặp lại (non-repeatable read) hoặc phantom hiển thị cho transaction, không phát hiện mọi serialization anomaly của các thao tác ghi trên các dòng khác nhau.

### Lock row của chính nhân viên

A và B vẫn lock các rows khác nhau. Quy tắc chia sẻ cần một guard chia sẻ hoặc lock toàn bộ tập hợp liên quan theo cùng một giao thức.

### Dùng unique/check constraint đơn giản

`CHECK` ở mức dòng không truy vấn được số lượng của các rows anh em. Ràng buộc unique không biểu diễn quy tắc “ít nhất một giá trị true”.

### Dùng `synchronized`

Không điều phối được các nodes ứng dụng khác, SQL từ admin hay tiến trình hàng loạt (batch process).

### Thử lại ngoại lệ optimistic (Retry optimistic exception)

Không có ngoại lệ nào xảy ra trong trình tự xen kẽ này. Việc thử lại cùng một quyết định mà không tải lại toàn bộ danh sách trực cũng không bảo vệ được quy tắc bất biến.

### Gửi thông báo trước commit

Nếu transaction sau đó bị rollback hoặc lỗi tuần tự hóa, thông báo trở thành một tác dụng phụ vô thừa nhận (orphan side effect). Hãy phát hành các sự kiện outbox bền bỉ trong successful transaction nếu cần thiết.
