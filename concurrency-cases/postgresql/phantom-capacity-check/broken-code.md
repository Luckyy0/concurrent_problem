# Mớ Code Đếm-Xong-Nhét Dễ Ăn Hành Mất Ghế (Broken count-then-insert allocation)

## 1. Cục Cấu Trúc Khối Bể (Pool entity)

```java
@Entity
@Table(name = "processing_pool")
public class ProcessingPool {
    @Id
    @Column(name = "pool_id", nullable = false)
    private UUID id;

    @Column(nullable = false)
    private int capacity;

    protected ProcessingPool() {
    }

    public UUID getId() {
        return id;
    }

    public int getCapacity() {
        return capacity;
    }
}
```

## 2. Cục Gạch Xếp Ghế (Allocation entity)

```java
@Entity
@Table(
    name = "slot_allocation",
    uniqueConstraints = @UniqueConstraint(
        name = "uk_slot_allocation_request",
        columnNames = {"pool_id", "request_id"}
    )
)
public class SlotAllocation {
    @Id
    @Column(name = "allocation_id", nullable = false)
    private UUID id;

    @Column(name = "pool_id", nullable = false)
    private UUID poolId;

    @Column(name = "request_id", nullable = false)
    private UUID requestId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private AllocationStatus status;

    protected SlotAllocation() {
    }

    public static SlotAllocation active(UUID poolId, UUID requestId) {
        SlotAllocation allocation = new SlotAllocation();
        allocation.id = UUID.randomUUID();
        allocation.poolId = poolId;
        allocation.requestId = requestId;
        allocation.status = AllocationStatus.ACTIVE;
        return allocation;
    }

    public UUID getId() {
        return id;
    }
}
```

Nút thắt Cấm Trùng `unique(pool_id, request_id)` chặn đúng việc gởi trùng 1 mã số 2 lần tạo ra 2 tờ vé. Chứ Thằng A Mã Số Bất Động Sản X, Thằng B Cầm Mã Căn Nhà Y, Khác Nhau Nên Thằng Khóa Trùng Câm Nín Lùi Lại Éo Đỡ Được Gì Bơm Hụt Ghế Sức Chứa Hết Nha Nàng!

## 3. Câu Thần Chú Bốc Thuốc Đếm Đủ Số Chưa (Repository predicate query)

```java
public interface SlotAllocationRepository
    extends JpaRepository<SlotAllocation, UUID> {

    @Query("""
        select count(a)
          from SlotAllocation a
         where a.poolId = :poolId
           and a.status = :status
        """)
    long countByPoolAndStatus(
        UUID poolId,
        AllocationStatus status
    );
}
```

## 4. Tên Lính Đưa Lệnh Vớ Vẩn Quá Mù Quáng (Broken service)

```java
@Service
public class PoolAllocationService {
    private final ProcessingPoolRepository pools;
    private final SlotAllocationRepository allocations;

    public PoolAllocationService(
        ProcessingPoolRepository pools,
        SlotAllocationRepository allocations
    ) {
        this.pools = pools;
        this.allocations = allocations;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public AllocationResult allocate(UUID poolId, UUID requestId) {
        ProcessingPool pool = pools.findById(poolId)
            .orElseThrow(PoolNotFoundException::new);

        // ĐỨNG LẠI, Đếm!
        long active = allocations.countByPoolAndStatus(
            poolId,
            AllocationStatus.ACTIVE
        );

        // PHÁN QUYẾT TO TÁT Ở ĐÂY NÀY!
        if (active >= pool.getCapacity()) {
            return AllocationResult.full();
        }

        // HẠ BÚT KÝ, NHÉT VÀO!
        SlotAllocation saved = allocations.save(
            SlotAllocation.active(poolId, requestId)
        );
        return AllocationResult.accepted(saved.getId());
    }
}
```

Nhìn code thì Trông Khá Quen Thuộc (realistic): Chỗ Check Sức Chứa Nằm Gắn Chặt Lì Cùng Lúc Chung Quản 1 Khung Với Cái Thằng INSERT Luôn Lại Cú Cộng Kép Ép Khóa Chặn Trùng Khách. NHƯNG PHẦN HỞ CHẾT RUỘT Là Quên Rào Xích Bảo Kê Khúc Phán Quyết Chung Giữa DB Lọc Điều Kiện Dòng Khống Lưng Aggregate Predicate Á!

> **Sếp chốt lại:** Có Cái Áo Transaction Giúp Cho Cái Phát Ngồi Đặt Ghế Lọt Mâm Đậm Đặc Liền Trái Atomic Thôi; CHỨ Hai Kẻ So Tài Đồng Diễn Vẫn Hoàn Toàn Tự Đắc Xem Chung Số Bàn 9 Rồi Ký Chết Đè Phóng Hai Tờ Chốt 10 Mảnh Nằm Cạnh Riêng Tách Kẹt!

## 5. Cuốn Băng Ghi Cảnh Thằng Hibernate Múa Máy SQL Ở Trong (SQL Hibernate thực thi)

Cả Cu A Lẫn Cu B Đều Hét Gọi Cùng Kịch Bản:

```sql
begin;

select p.pool_id, p.capacity
from processing_pool p
where p.pool_id = :poolId;

select count(*)
from slot_allocation a
where a.pool_id = :poolId
  and a.status = 'ACTIVE';
-- Hai Bé Đều Há Mõm Nhìn Cùng 1 Con Số 9

insert into slot_allocation(
    allocation_id,
    pool_id,
    request_id,
    status
)
values (:differentAllocationId, :poolId, :differentRequestId, 'ACTIVE');

commit;
```

Cục Vàng Hibernate Nó Hay Ưa Dìm Giữ Cú Xóc `INSERT` Chờ Đến Chốt Sổ Xả Áo Ống Flush/Commit Mới Phóng DB Nghe. Nhưng Gắng Ép Nhả Mở Van Trượt `flush` Sớm Ra Liền Lại Càng Chỉ Chắp Cánh Cho Dòng Mới Rõ Sáng Đánh Lên Cho Mình Tự Nhìn Sau Tịch Commit, Chứ ĐÉO Gài Nổi Đi Cắn Móc Lấy Lưới Khoảng Rỗng (predicate gap) Bịt Mắt Chống Cái Gậy Kẻ Thù Kéo Nhét Nhá!

## 6. Dây Dưa Va Đập Ngẫu Nhiên Lên Sàn Đấu (Concrete interleaving)

```text
A MỞ RÀO BEGIN
B MỞ RÀO BEGIN
A QUÉT MẮT ĐẾM VÉ ACTIVE -> Lọt Dõng 9
B QUÉT MẮT ĐẾM VÉ ACTIVE -> Dính Số 9
A Ký Sổ Chuẩn Bị Nhét A-101
B Ký Sổ Chuẩn Bị Nhét B-202
A Buộc Bụng Ói Ra Lệnh Khạc INSERT A-101 -> Đậu
B Bấm Bụng Xả Cống Khạc Lệnh INSERT B-202 -> Đậu
A VANG BÚA CỤP COMMIT
B VANG BÚA CHÉM COMMIT
Tổng Kết Tàn Khốc Đếm ACTIVE Kéo Cuối Dây -> 11 CÁI ĐẦU ĐU MÁ!!!
```

Bởi Hai Dòng Lệnh Rõ Khác Key Tươi Tốt Bám Phao Đứng Im Á, Cục Rào Chặn Ác Thề Nào Quản Soi Máu Cắt Khóc Được Nghĩa Sức `COUNT(ACTIVE) <= capacity`.

## 7. Mở Mắt Thấy Bóng Ma Đứng Chờ Ngay Giữa Chợ (Visible phantom khi query lại)

Trực Thuộc Vòng Cách Ly Quần `READ COMMITTED` Lọt Tay Đi Có Khả Năng Tự Trông Thấy Hiện Tượng Rùng Rợn:

```sql
select count(*) ...; -- Mới dòm Số 9
-- Thằng B Quật Cái Kéo Giật Thọc Đáy Nhét Xong Khóa Sổ Cụp Xuống (commit)
select count(*) ...; -- Nhe Răng Rụng Xún Thành 10
```

Phát Nhìn Gọi `SELECT` Vét Sau Nắm Chụp Tấm Ánh Đèn Bắn Tia Snap Mới Hơn Nên Thấy Ma Hiện Ra (visible phantom) Đó Nghen. NHƯNG Cái Kịch Bản Nát Phím Broken Service Đời Lại Gấp Nát Rối Nùi, Nó KHÔNG Rảnh Việc Chọt Quay Lại Dòm Đếm Lần 2 Cho Nát; CHỈ MỚI Tự Sướng Khựng Óc Dựa Lời Count Lỗi Lầm Mốc Cũ Kéo Ánh Là Đủ Ép Thần Tranh Dành Vào Gài Chốt Xí Chỗ Check-Then-Insert Đi Tử Bỏ Rồi Dô!

## 8. Trùm Kính Lên Khung REPEATABLE READ Vẫn Rớt Bịch Bịch Chết Nữa Đâu Khỏi Trốn! (Tại sao `REPEATABLE READ` chưa đủ)

Cả Cu B Kéo Kính Cách Ly Sạch Dỏm Đổ Bức Tường `REPEATABLE READ`, Trấn Che Bịt Sóng 2 Cú Mắt Máy Đứng Im Vẫn Trông `9`.
Hai Móc Nhét Phóng Xuống Cùng Bậc KHÔNG Đánh Ngang Sọc Đồng Dòng Nhau:

```text
Chụp Màn Hình Của A: Vẫn Đếm 9 -> Sướng Đi Quật INSERT A
Chụp Màn Hình Của B: Dính Chữ 9 -> Mở Cửa Vui Kép B Nhét
CẢ HAI CỨ THẾ BẤM XÉO LỌT COMMIT TỈNH BƠ -> Vươn Cột Quá Bờ Rào Ra Tận Số 11
```

Chú PostgreSQL Mới Thò Tay Ngăn Đạn Áo Khói "Bóng Ma Nhấp Nhô Khi Bị Đếm Trực Trở Lại Dài" Ở Tầng Đệm Vát Bọc Kia, Lòng NÓ ĐÉO Ban Kèm Sóng Khóa Chống Trọi Viết Giết Bịt Lấp Tranh Chỗ Khống Cửa Có Chứa Dụng Ý Cao Sang Chỗ! Kẹp Ảnh Sống Trượt Bật Lão Hóa Ổn Định Tĩnh Im Gà Quán (Stale snapshots) CHỈ Đẩy Mọi Kẻ Khờ Gối Mơ Tưởng Yên Bình Rằng Kho Ổn Lắm Bố Mày Cứ Thế Đếm Vững Số 9 Nhé Lọt Thỏm Đít!

## 9. Chỉnh Thông Số Đáy Để Rập Khuôn Mời Lỗi Ra Chụp (Preconditions tái hiện)

1. Mâm Bể Gọn Khúc Chuẩn Bị Tịch Mức Phình Đầy Khấc `capacity - 1`.
2. Hai Lính A Và B Ôm Lệnh Đi Cổng Rời Xài Bơm Móc Cống 2 Sợi Giao Dịch Cháy Song Ngược Riêng.
3. Kịp Nổ Đếm COUNT Của Cả Nhóm Ngang Tới Điểm Trước Kịp Khi Hòn Đá Chốt Áo INSERT Được Gõ Chuông Nhắm Rụng Xéo.
4. Mác Trụ Danh Allocation/Request IDs Điểm Riêng Tuyệt Đối Khác Phá Nhau Nhé.
5. Sân Khấu Sạch Trơn Mù Không Có Rào Gọi Khóa Hộ Ngự Hàng Bố Mẹ, Cu Máy Cộc Đếm Khúc Liệt Hay Lưới Viền Đẩy Retry Cột Chuẩn Chết SSI Nào Che Xéo.
6. Thằng Múa Test Ở Ngự Cửa Hàm Gọi Chạy Đ ÉO Tặng Phũ Cho Áo Lông Choàng Transaction Đè Trùm Kẹp Cáo Giao Khung Các Cú Chốt Ẩn Commits Trơn Nheo.
7. Đút Trực Mâm Cựa Trạm PostgreSQL Lên Nhịp Sàn Tránh Bịp Để Phô Rõ Trói MVCC/Isolation Quất Áp Chết!

## 10. Liều Thuốc Gà Mờ Mắc Lỗi Áp Vào Nào Có Phê (Những cách sửa chưa đủ)

### Dán Trộm Miếng Dán `@Transactional` Thôi Chưa Đủ Ấm Cúng Gì

Áo Bọc Ráp Quấn Tròn Cũng Vướng Tơ Dở Tình Rời Hạt Khi Trái Óc Phán Kéo Giọng Dòng Điều Kiện (Non-atomic predicate decision) Bắt Lại Kéo Trượt Qua Ba Bốn Căn Dây Lệnh Rời.

### Tin Tưởng Tấm Lưới Unique Đè Đầu Căn Cước Allocation ID/Request ID Khùng Trí Nữa

Cái Còi Hú Này Gọi Chặn Cắt Cột Trụ Đứng Dấu Điên Xé Tới Mọc Cọc Trùng Thôi Mã (Duplicate identity), Không Siết Tức Khoán Giới Kéo Cố Thòng Cụt Đứt Rễ Số Giống Row Phán Quyết!

### Cúi Đầu Xin Lấy Khóa Phủ Hết Đám Tồn Tại Trong Phòng (Lock các rows hiện có)

Sủa Ùm Ùm Quát Gắn Khóa Chốt Kẹp Mấy Đứa Vé Cũ Thì Cu Kéo Kém Đuôi Kia Lặng Đi Mò Quăng Một Con Dòng Bất Đắc Dĩ Mới Ra Mà Nào Đâu Sờ Đuôi Ẩn Nó Trước Lọc Trừng Dày Gắn Lỗ Đâm. Hú Khét Câu Rỗng Đếm `COUNT(*) FOR UPDATE` Á? Nó ÉO Bào Giờ Lãnh Án Đọc Khoá Gìn Ép Một Chó Cái Vùng Chữ Tĩnh (predicate range) Dài Trên Gốc Rắn Cửa Postgres Đâu Á!

### Quẩy Chắn Cứng Quần Kéo Quát Nâng Tầm Lên Chốt `REPEATABLE READ` Đi 

Giương Mắt Rào Kép Kết Khóa Màn Ổn Đít Nhìn Trượt Khách Đứng Lặng Nhưng BẤT LỰC Chấp Không Đẻ Phát Đánh Sóng Sụt Dội Quật Conflict Ở Kẽ Thở Nhét Giữa Cháy Hai Đợt Inserts.

### Trốn Rừng Mò Xài Áo Đi Đêm Tôn Nham `synchronized` Cùi Nhảy

Rúc Rèm Quanh Lẩn Trốn Nép Góc Quanh Quẩn 1 Góc Xóm Phố (JVM) Lọ Mọ Gánh Xéo Bọc Code Khép Tròng Quên Át Máy Tách Cửa Rào Nước Ở Kênh Thể Trạm Khác Ập Admin Tool Trực Máy Batch Lôi Chéo Kể (Không tham gia cùng monitor). Ngu Vừa!

### Hét Quát Sủa Nhả Xả Xống Liền Oai Tuấn (Flush trước khi trả result)

Ép Trút Ộp Khủng Xéo Mũi Flush May Kéo Cửa Vỡ Phọt Trực Nứt Lòi Sóng Gấp Violation Chứ Làm Khô Trống Ế Có Cái Khung Móc Ép Thép Bịt Ngạnh Capacity Nào Lồi Tờ Vi Phạm Che Nữa Trật Quần Đâu. Mắt Khách Sang Bờ Bên Muốn Ngó Kép Quả Động Trời Lộ Nhét Phải Chờ Về Lệnh Trảm Chốt Bơm Nằm (commit).

### Xòe Tay Xin Cấp Uống Lặp Bù Khốc Cạn Retry Loạn Xới Chọt Đám Exception

Vũ Điệu Kép Xung Phá Này KHÔNG Éo Chọt Mỏi Tiếng Oán Của Nút Báo Quẳng Khóc Exception Ở Bức Cửa Ảo Dễ Quẹt `READ COMMITTED` Lọt Đâu. Gọi Hơi Ép Đập Uống Trống Không Tích Gọi Đơn Kép Idempotent Rác Mùi Thì Còn Tạo Ọc Bơm Dọc Phọt Ghi Bãi Dày Chặt Cứt Effects Tàn Hoang Kia Nha!
