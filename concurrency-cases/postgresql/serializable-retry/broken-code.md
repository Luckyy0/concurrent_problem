# Code Bể Trầm Trọng — Ngộ Nhận `SERIALIZABLE` Là Tấm Bùa Bất Tử (luôn thành công)

## 1. Dọn Khung Bàn Nháp (Schema tối thiểu)

```sql
create table merchant_limit (
    merchant_id bigint primary key,
    limit_amount numeric(19, 2) not null check (limit_amount >= 0)
);

create table credit_reservation (
    reservation_id uuid primary key,
    command_id uuid not null unique,
    merchant_id bigint not null references merchant_limit(merchant_id),
    amount numeric(19, 2) not null check (amount > 0),
    status varchar(16) not null check (status in ('ACTIVE', 'RELEASED'))
);

create index ix_credit_reservation_scope
    on credit_reservation(merchant_id, status);

create table limit_command_decision (
    command_id uuid primary key,
    merchant_id bigint not null,
    requested_amount numeric(19, 2) not null,
    outcome varchar(16) not null check (outcome in ('ACCEPTED', 'REJECTED'))
);

insert into merchant_limit(merchant_id, limit_amount)
values (7, 100.00);

insert into credit_reservation(
    reservation_id, command_id, merchant_id, amount, status
)
values (
    '00000000-0000-0000-0000-000000000060',
    '10000000-0000-0000-0000-000000000060',
    7, 60.00, 'ACTIVE'
);
```

Cái Index này kéo cho cái mắt soi access (predicate) rõ chữ nét thực tiễn hơn xíu. Cơ mà bộ SSI vẫn có quyền lấy áo lưới page/relation bọc `SIReadLock` rải xuống bất cứ lúc nào tuỳ tâm cái execution plan của nó nha; Viết Code CẤM có đi đánh đu dựa dẫm vào ba cái vỏ bọc mỏng manh (lock granularity) thay đổi xoành xoạch này!

## 2. Kho Dữ Liệu Rờ Số (Repository đọc predicate)

```java
package example.limit;

import java.math.BigDecimal;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

public interface CreditReservationRepository
        extends JpaRepository<CreditReservation, java.util.UUID> {

    @Query(value = """
            select coalesce(sum(amount), 0)
            from credit_reservation
            where merchant_id = :merchantId
              and status = 'ACTIVE'
            """, nativeQuery = true)
    BigDecimal activeTotal(@Param("merchantId") long merchantId);
}
```

Mấy cái Repository ruột thịt đi kèm:

```java
public interface MerchantLimitRepository
        extends JpaRepository<MerchantLimit, Long> {
}

public interface LimitCommandDecisionRepository
        extends JpaRepository<LimitCommandDecision, UUID> {
}
```

Phần Entity cứ ráp trúng khuôn đúc DDL là bao xài. Chỗ `CreditReservation.accepted(...)` với `LimitCommandDecision.accepted/rejected(...)` thực ra chỉ là ba cái lò nặn Factory nén nhào cục entity mới toanh nhưng vẫn ngậm khư khư cái mã `commandId` cũ thôi hà.

## 3. Khứa Cởi Trần Phá Lưới (Attempt có isolation đúng nhưng thiếu failure contract)

```java
package example.limit;

import java.math.BigDecimal;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class BrokenSerializableLimitService {

    private final MerchantLimitRepository limits;
    private final CreditReservationRepository reservations;
    private final LimitCommandDecisionRepository decisions;

    public BrokenSerializableLimitService(
            MerchantLimitRepository limits,
            CreditReservationRepository reservations,
            LimitCommandDecisionRepository decisions
    ) {
        this.limits = limits;
        this.reservations = reservations;
        this.decisions = decisions;
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public LimitDecision reserve(ReserveLimitCommand command) {
        var replay = decisions.findById(command.commandId());
        if (replay.isPresent()) {
            return LimitDecision.from(replay.orElseThrow());
        }

        BigDecimal limit = limits.findById(command.merchantId())
                .orElseThrow()
                .limitAmount();
        BigDecimal active = reservations.activeTotal(command.merchantId());

        if (active.add(command.amount()).compareTo(limit) > 0) {
            decisions.save(LimitCommandDecision.rejected(command));
            return LimitDecision.rejected(command.commandId());
        }

        reservations.save(CreditReservation.accepted(command));
        decisions.save(LimitCommandDecision.accepted(command));
        reservations.flush();
        decisions.flush();

        return LimitDecision.accepted(command.commandId());
    }
}
```

Hàm này mà chạy cô đơn 1 mình thì ra quyết định (decision) chuẩn men lắm. Kẹt cái là tụi API/Caller cứ đinh ninh nhắm mắt tin tưởng cái lò này LUÔN nhả cục `LimitDecision` thơm tho béo ngậy! Nếu nhét 2 mũi khoan rặn (attempts) cùng cắm song song, cái nhát súng `40001` có thể xé gió bật ra ngay ở chỗ query, xịt trào lúc flush hoặc ngay đáy búa đập cuối transaction commit, mặc dù cái bụng ruột hàm (method body) nó rặn đẻ ra mẻ return value rồi á!

Bữa mà exception bật trúng điểm commit ấy, mày có lấy rổ `try/catch` úp chụp bao bọc cái cục `flush()` lọt bên trong method thì đéo đời nào vớt được ẻm! Bởi tụi Transaction Interceptor Đứng Mâm Mũ Tầng Ngoại mới là bọn nắm đinh búa chốt nhịp commit cuối cùng.

> **Sếp chốt lại:** Giăng cái mác nhãn isolation thì có thể dán bùa trúng đó, nhưng ván use-case của mày vỡ nát Bét Sứ Correctness nếu éo vạch sẵn giao kèo đứt đuôi (failure contract): Thằng nào chết Rollback ra sao, Kéo Đấm Lại (retry) chỗ nào, Mức Nào Kiệt Thở Ngáp Gãy (exhaustion) Phải Buông!

## 4. Cái Mõm Loa HTTP Đóng Khét Lỗi Database (Controller làm rò lỗi kỹ thuật)

```java
@RestController
public class LimitController {

    private final BrokenSerializableLimitService service;

    public LimitController(BrokenSerializableLimitService service) {
        this.service = service;
    }

    @PostMapping("/merchants/{merchantId}/reservations")
    ResponseEntity<LimitDecision> reserve(
            @PathVariable long merchantId,
            @RequestBody ReserveRequest request
    ) {
        return ResponseEntity.ok(service.reserve(request.toCommand(merchantId)));
    }
}
```

Nộp Lệnh chọt ngáp rớt bịch trái đạn thối HTTP 500 mặc kệ PostgreSQL bọc lưới rollback êm ru tuốt lọt rạch kẽ (rollback sạch). Client ăn đòn sập thì nổi khùng nhắm mắt đẻ command ID MỚI ráng chọt lại (retry), thế là cấy trùng lắp Operation rác (duplicate operation) làm giòi bọ lỗi gãy kẹp nhau (conflict rate) tăng nổ nóc!

## 5. Rặn Sai Rốn Ục Lại Quanh Cái Đáy Mâm Vỡ (Retry sai trong cùng transaction)

```java
package example.limit;

import jakarta.persistence.EntityManager;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class BrokenLoopRetryService {

    private final SerializableSteps steps;
    private final EntityManager entityManager;

    public BrokenLoopRetryService(
            SerializableSteps steps,
            EntityManager entityManager
    ) {
        this.steps = steps;
        this.entityManager = entityManager;
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public LimitDecision reserve(ReserveLimitCommand command) {
        for (int attempt = 1; attempt <= 3; attempt++) {
            try {
                return steps.decideInsideCurrentTransaction(command);
            } catch (RuntimeException failure) {
                if (!PostgreSqlFailures.isSerializationFailure(failure)) {
                    throw failure;
                }
                entityManager.clear();
                // TÈ TÈ TÈ Sai Bét: Cuộn database transaction và bảng snapshot cũ rích rỉ máu vẫn đang nằm thở hóp failed kìa nhóc.
            }
        }
        throw new LimitContentionException(command.commandId());
    }
}
```

Bị táng cục `40001` vỡ đầu, ván cờ PostgreSQL transaction chìm ngỉm lụt chìm Failed state. Bất cứ phát đâm Query tiếp theo nào cũng dính rạch ngực trét bùa ức `25P02` cho tịt đứt tới khi nào hít lệnh `ROLLBACK` mới ngáp được. Vung chổi `EntityManager.clear()` nó chỉ phẩy nhẹ quét đám rác bọt Java entities thôi (detach); Đéo hề có dụ nhổ gốc Rollback connection, đéo xé nháp Bàn Snapshot cờ mới toanh, càng không hốt cứt bôi xóa được mẻ Đòn Phạt Oái Oan Chóp External Side Effect Bọc Lại (side effect)!

Dẫu mày có mưu chước lôi chốt giăng neo savepoint ra đỡ, Bọn Già Làng PostgreSQL nó vẫn ép mày cởi áo trần đấm rạch Rã Trọn Giao Dịch Sạch Mẽ (retry complete transaction) Đẻ Lại Từ Đầu cho cục u serialization failure; Retry mót chẻ mảnh mụn (fragment) thì Cứt Trôi Đi Sao Trôi Hết Căn Logic Rặn Ra Đứa Ngậm Chốt SQL và Mớ Số Má kia!

## 6. Lộn Rễ Nấm Bám `@Retryable` Trùm Đầu Chung Nhánh Tụt Vòng `@Transactional`

```java
@Retryable(
        retryFor = RuntimeException.class,
        maxAttempts = 4
)
@Transactional(isolation = Isolation.SERIALIZABLE)
public LimitDecision reserve(ReserveLimitCommand command) {
    return decide(command);
}
```

Đeo gông ngửa 3 vố dọng đầu:

1. Trùm mền `RuntimeException.class` là nó bọc bắt vác rặn Lại (retry) Cả lũ Ngáo Hút Phọt Input Lỗi Bừa Bãi Hay Gãy Code Ngu Nhỏ Dãi (validation/programming failures) cmnl!
2. Cái lũ Kháo Trọng Áo Giáp Lệnh (Retry và transaction advisors) đè nhau nằm ngợp tròng trên Đỉnh 1 Cây Proxy khiến Ngõ Cửa Ranh Giới Boundary Trắng Tay Lệ Thuộc Bừa Vào Bọn Advisor Chồng Thứ Tự Rúc (ordering/configuration).
3. Đéo vạch Mức Tuột Trào Deadline Hủy Ván Tổng Bọn (overall deadline), Đéo Hút Lệnh Chia Phân Loa SQLSTATE Chuẩn Lép Chẻ Đinh Cắm Khóa Chốt (durable command replay) Thét Lại Ngáp!

Nếu tấm Lệnh Bọc Transaction bao trùm đít nhét Lệnh Bóp Rặn (retry advice) ngộp bên trong, mấy đợt Tái Chạy Khẽ Dọng Chết Nghẹt (attempts sau) Cứ Trượt Thẳng lọt vào trong bọc mâm chung Máu Trào Vỡ Đuôi Doomed cmn ròi. TÁCH NÁT Trạm Mở Gọi Coordinator Văng Ra Dời Đóng Bọn Bóp Thợ Worker beans Thành Tuyến Ranh Giới (boundary) Cho Trắng Sáng Mặt Mày Và Đâm Test Mượt Hơn Nhé Cưng.

## 7. Trò Áo Giáp Chọc Hút Sóng Mù Tự Trọng Rốn Tự Khóa (Self-invocation làm mất isolation)

```java
@Service
public class BrokenFacade {

    public LimitDecision reserve(ReserveLimitCommand command) {
        return serializableAttempt(command); // KÉO RỐN THÉT TỰ CHỌC LÙI this.serializableAttempt(...)
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public LimitDecision serializableAttempt(ReserveLimitCommand command) {
        // read → decide → write
    }
}
```

Trò Hô Gọi Tiếng Hú Rừng Trong Nội Bộ lủng đéo thể xé tròng Mặt Kính Spring Proxy. Nếu Đéo mang Mâm Transaction ở Bìa Vòng Ngoài bao ốp, Lũ Trọc Ngoáy Repository Calls có thể rách lở bừa bãi lôi ra các transactions cọc cạch đục rời nhau vọt xéo (hoặc ngậm rốn default `READ COMMITTED`); Cón Nếu Mà Đã Mang Mặt Lót Áo Cũ Outer Transaction Ổn Ôm Đã Có Ròi, Mớ Chữ Bọc Mác Annotation Bịt Trong (inner) BẤT LỰC Ép Độ Độn Isolation Kép Gồ Bật Trồi Kín Rìa Mặt Bệ (không nâng effective isolation)!

## 8. Loa Phóng Tin Gửi Khách Gãy Vỡ Nát Trước Lọc Commit (External side effect)

```java
if (allowed) {
    reservations.save(CreditReservation.accepted(command));
    notificationClient.sendAccepted(command.commandId()); // Chết Đóng Định Cứt Trào không rollback được Lôi Lại Đâu!!!
    return LimitDecision.accepted(command.commandId());
}
```

Tụi Máy Xăm SSI Chốt Xéo Phọt Đạn Sụp Nhót Lúc Điểm Búa Khép Giao Dịch Dồn Ngược Ngực Commit SAU KHI Cánh Cửa Chữ Loa Khách Hàng (notification) Xé Mạch Bay Tung Quét. Bấm Nút Đấm Lại Cụ Dọng Nhát Khứa Thông Điệp Kẹp Gọi Lần Hai Nhồi Trùng Hoặc Bỗng Nhiên Lão Cú Chốt Cuối Khước Nặn Đỉnh Mới Phọt Rớt Máng Ra Vỡ Mặt `REJECTED`, Thế là Khứa Ôm Một Quả Bom Tréo Mép Giữa Đít Database Lọt Rọt Dưới Ục Khách External system Nhé!

## 9. Sân Khấu Kéo Xếp Mô Hình Kẹp Hiện Trường (Điều kiện để tái hiện)

1. Gieo Hạt (Seed) mốc limit `100` và bọc active chốt đổ ngòi trót lọt `60`.
2. Kép T1/T2 Kéo Lưới physical transactions riêng Ốp Kép Cách Ly Áo Bọc `serializable` Đều Đồng Bọn.
3. Cả 2 Phe Dính Ngậm Hút Vực stable snapshot Lút Ốc Cuộc Phán Dứt Bảng `SUM=60` TRƯỚC LÚC Nặng Lệnh Ghi Nhét (insert) Lọt Lòng Cửa Nhé!
4. Kẹp Thét Khứa C1/C2 Treo Đinh command/reservation IDs Riêng Giấu Trắng!
5. Bật Hàng Rào Chắn Trận (Barrier mở) Để Cả 2 Lên Đạn Ục (insert) Rồi Sút Kéo Búa Tịt Ngáp Commit!
6. Bãi Test Đào Bọc Om Trọn Rổ Exception Tuốt Vòng Viền Chẻ Kín Transaction, kể cả lúc móc sụt `commit()`.
7. Dọng Kho Thực PostgreSQL Chân Lực Ngạo Nghễ Mới Vắt Kéo Hàng Qua Testcontainers Nhé; Trò Con Nít H2 Nó Đéo Có Sân Phơi SSI/`SIReadLock` Cứt Chó Gì Rặn Soi Đâu!

## 10. Mớ Khứa Bó Thuốc Gãy Xương Dán Mù Éo Đủ Lệ (Các cách sửa chưa đủ)

- Chỉ Hét Đẩy Lên Cấp Kéo Cứng Isolation Mù Mờ Bỏ Rọt Lơ Xéo Mạng Cục `40001` Bơm Xịt.
- Nhét Rặn Retry Ở Đoạn Thằng Nháp Cuối Tít Góc Rạch Statement Đéo Lột Sạch Giao Trọn Vẹn Dịch Cục Mới Rút (whole transaction).
- Sớt Máng Mót Liếm Đỉnh Bụng Màng Entities / Mảng Khóa Ngáp Snapshot Tụ Lấy Lên Từ Rác Ải Failed Cũ (failed attempt).
- Ói Dồn Rặn Cáo Lặp Mãi Mép Nháp Vô Hạn Hay Vừa Đập Gãy Lại Khởi Đạp Ụp (ngay lập tức) Thiếu Máng Rụt Nháp Random Jitter Trễ Trút Nhịp.
- Cứ Vấp `DataAccessException` Ngậm Oải Lại Kéo Retry Bất Chấp Sọc Bọn Sụp Mặt Nặn Lá Cứt Code Lỗi Cả Nhau.
- Chế Phọt Cống Kéo Rạch Mã Đinh Command ID MỚI SỤT Bịch Khứa Áo Nháp Attempt Khắc Lệ Oác!
- Đẩy Phọt Loa Nháp Gửi Lọt Rút Dây HTTP/message Hét Sắp Trước Lúc Kẽ Khách Ánh Commit MÀ Éo Cáp Lọt Phễu Bồn Tụ Hàng Trực Outbox!
- Bịt Xẹo Áo Bọc Nốt JVM `synchronized` Rìa Mâm Đất Lọt Bụng Gáy Chắn Đơn Chút Éo Nhéo Cựa Mớ Kéo Hai Đội Hàng application instances 2 Phe Nắm Sừng Cổ Lock Khác Gãy Nhau!
- Lếu Láo Ngậm Ngóng Suy Bừa: Đứa Ngã Chết Tội (victim) BAO GIỜ Chả Là Khứa Chạy Tới Theo Chân Sau Nhỉ Lều (bắt đầu sau)!
- Hú Chóp Rờ Nắm Xoắn Khứa Đầu Dọng Mõm Test Assert Chỉ Nhăm Type Bọc Máng Áo Bệnh MÀ Éo Cạp Quét Tóm Kép Xem Con Số Final Total Vẫn Giữ Vững Họng Oanh Quyết Mép Outcome Đúng Đẹp Decisions!
