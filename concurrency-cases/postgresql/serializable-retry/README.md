# Cảnh Báo DB-009 — Chơi Hệ Khóa Đỉnh Bảng `SERIALIZABLE` Và Thuốc Giải Dựng Lại Chết Đi Sống Lại (Retry an toàn)

## 1. Mạch Chuyện Tóm Lược (Tóm tắt)

Thử vẽ cảnh này nha em: Một tay buôn có mức trần đặt cọc (reservation limit) là `100`. Tổng số tiền đang bị găm (active) là `60`. Rồi bùm, 2 ông thần giao dịch (command) bay vô giành giật, đòi cọc thêm `30` mỗi ông:

```text
Anh T1 móc túi thấy tổng 60 → gật gù DUYỆT (ACCEPTED) → cắm xuống cọc C1.
Anh T2 cũng thò tay đo được 60 → phán DUYỆT (ACCEPTED) → cắm mốc cọc C2.
```

Bỏ mẹ! Nếu cả hai gã đều đóng búa chốt đơn (commit), tổng vọt mịa lên `120`, vỡ trần cháy túi!
Lúc này, Cảnh Sát Trưởng PostgreSQL bận áo `SERIALIZABLE` xách súng **Serializable Snapshot Isolation** (`SSI`) ra đường tuần tra các nhánh đọc/ghi (read/write). Chừng nào ổng phát hiện cái mớ bòng bong đồng thời này (concurrent history) KHÔNG THỂ vạch ra được một đường chạy nối đuôi nhau ngay ngắn (tuần tự), ổng sẽ bắn vỡ sọ (abort) 1 thằng với mã lỗi khét lẹt: SQLSTATE `40001`.

Tức là áo giáp `SERIALIZABLE` chỉ bảo chứng cho đám **đã sống sót qua ải commit**, chứ đéo rảnh hứa bao thầu mọi thằng chui vào đều sống nhăn! Nhiệm vụ của em (Application) là phải biết gom xác cuộn băng (rollback), lui bước dưỡng sức (backoff) rồi quăng 1 thằng đệ MỚI TINH ôm luật cũ vào đấm lại (transaction mới)!

> **Sếp chốt lại:** `SERIALIZABLE` là chiêu biến cái lỗ hổng ảo giác (anomaly) thành một cú tát chết có chủ đích (failure có kiểm soát); Chỗ em khoanh vòng Dựng Lại (retry boundary) mới thực sự nặn cú tát đó thành kết quả chốt sổ ngon ơ!

## 2. Diễn Viên Trực Chiến Và Đồ Bạc Dùng Chung (Actor và trạng thái)

| Cục Diện Lõi (Thành phần) | Hiện Trạng Đang Cầm (Trạng thái) |
| --- | --- |
| `merchant_limit` | Bác merchant `7`, ôm limit `100` |
| `credit_reservation` | Đang có sẵn cọc `ACTIVE` kẹt `60` |
| Bác T1 cắm trên App-1 | Tờ Lệnh C1, đòi cắn `30` |
| Bác T2 dậm trên App-2 | Tờ Lệnh C2, cũng xin `30` |
| Ngân Hàng Cha Giữ Sổ (Authoritative store) | Bác Bảo Vệ PostgreSQL |

Điểm chém lộn KHÔNG PHẢI là giành xé chung 1 mẩu thịt (row) update đâu nha. Cả 2 lão cùng soi chung 1 cái lỗ kính đo số (predicate):

```sql
select coalesce(sum(amount), 0)
from credit_reservation
where merchant_id = 7
  and status = 'ACTIVE';
```

Hút mắt xong mỗi lão nặn ra đút xuống 1 tờ cọc HOÀN TOÀN KHÁC NHAU. Nhưng ác nỗi cái tờ mới lọt vào lại chọc mù luôn cái lỗ kính số tổng mà lão kia vừa soi trước đó!

## 3. Khung Sườn Chống Đạn Bất Di Bất Dịch (Invariant)

```text
Tổng số tiền ôm cọc ACTIVE cho một nhà buôn tuyệt đối KHÔNG ĐƯỢC trào qua vạch limit đã chốt sổ.

Mỗi tờ lệnh ID chỉ mang ĐÚNG 1 mạng chốt kiên định (durable decision): ĐƯỢC CẤP (ACCEPTED) hoặc CÚT (REJECTED).

Bất cứ ván đấm lại (retry) nào bắt buộc phải chạy vắt lại trọn luồng Đọc → Chốt → Ghi trong bọc Giao Dịch MỚI TINH; KHÔNG để lọt rác rưởi của kiếp trước đã đổ bể làm dấy mùi!
```

Giả sử thằng T1 nó chốt trót lọt (commit) trước. Cú tái sinh MỚI NGON của T2 lúc này hít số đo thấy vọt lên `90`, nó tự cắn rứt nhận ra `90 + 30 > 100` rồi phán đinh ghim `REJECTED`. Kết cuộc bàn cờ sạch đẹp: mốc `90`, C1/C2 mỗi thằng an vị đúng 1 cái phán xét kiên định. Vỗ tay!

## 4. Ranh Giới Kép Vòng Đấu Trụ (Ranh giới transaction)

Thằng Dẫn Đuổi Đấm (Retry coordinator) ở trên TẦNG NGOÀI đéo có áo khoác Giao dịch gì đâu nhen. Mỗi vòng thét gọi rớt xuống trạm Proxy `SerializableAttemptService` mới là 1 bọc kén thực địa chém nhau đàng hoàng:

```text
Mạng số N
  BẮT ĐẦU VỚI GIÁP SERIALIZABLE
  xem tờ lệnh ID này đã sống chưa
  ngó trần tiền limit và đong tổng active
  chốt sổ (decide)
  nhét lệnh reservation nếu được gật
  khắc thẻ decision cho command
  dội bùn hất (flush)
  THẮNG (COMMIT) hoặc ĂN BẠN 40001 CHẾT CUỘN LẠI (ROLLBACK)
```

Chuyện đụng độ có thể bật lửa ở lúc ném lệnh câu (statement), lúc trào bùn Hibernate flush hay ngay phút chót đập búa commit. Thằng Dẫn Đuổi ở ngoài chỉ bắt hứng xác SAU KHI Lò Proxy kia cuộn sạch sẽ ván dập (rollback). Nó KHÔNG chơi trò ném vá vài câu SQL mót nhặt hay xài lại cái Máng Trộn Chết `EntityManager`, không xài xác rác entities vương vãi của kiếp trước!

## 5. Mộng Mơ Và Ác Mộng Cắt Dây (Kết quả mong đợi và thực tế lỗi)

| Khúc Đo (Khía cạnh) | Sếp Phán Nhẹ Nè (Mong đợi đúng) | Trẻ Trâu Tưởng Bở (Cách hiểu sai) |
| --- | --- | --- |
| Lưới Bọc `SERIALIZABLE` | Mớ Bòng Bong Lọt Lưới Bằng Đúng Nghĩa Bước Tuần Tự (Commit history tương đương chạy tuần tự) | Đứa Nào Cũng Chờ Nhau Ngáp Rồi Chốt (Mọi transaction tự block rồi commit) |
| Chết Phọt Huyết `40001` | Lỗi Bắt Bài Chôn Ngay, Trải Mới Luôn Mạng (Known rollback, có thể whole-transaction retry) | Trận Database Sập Nhảm, Ịm Cho Qua Tịt (Lỗi database bất ngờ hoặc có thể nuốt) |
| Ván Tái Đấu (Retry) | Xé Nháp Dựng Transaction/Snapshot Mới Rờ Xét Phép MỚI LẠI TOÀN BỘ (Transaction/snapshot mới) | Chơi Vòng Tròn Luẩn Quẩn Trong Cùng 1 Hàm Khoác `@Transactional` |
| Luật Bám Dính Nhẹ (Idempotency) | Giữ Đít Thẻ Áo Command ID Dũa Dáng Durable Trọng Lại | Tung Đấm Lại Ném Phọt Command ID Khác Nhé! |
| Sứt Dây Dẹp Tuổi (Exhaustion) | Ép Thất Bại Rõ Ngắn Hẹn Kẹp Gấp (Trả explicit temporary failure) | Rặn Ục Ịt Vô Hạn Giết Nhau Chờ Oái (Retry vô hạn tới khi success) |

## 6. Lổ Tai Nghe Thuật Ngữ (Thuật ngữ cần biết)

| Chữ Đúc (Thuật ngữ) | Giải Mã Trận Tranh Này (Ý nghĩa trong case) |
| --- | --- |
| Súng Kẹp SSI | Trọng Nhịp Serializable Snapshot Isolation; Bóng Ma Snapshot Xịn Ốp Lưới Đo Tranh Chéo (dependency checks) |
| Phá Lưới Đứt Cuộn (serialization failure) | PostgreSQL bắn rớt do mớ lộn xộn đéo thể xếp hàng trật tự (history không thể serialize) |
| Lỗi Tụ Huyết SQLSTATE `40001` | Mã Thẻ Chết Khét `serialization_failure` |
| Lưới Bắt Bọng (predicate lock) | `SIReadLock` quăng trói canh chừng mảng số đếm/chữ lọc; KHÔNG phải sợi xích row đứng nghẽn cứng ngắc! |
| Lấn Cửa Nhau (rw-conflict) | Mày Dòm Xoi Tờ Kính Mà Đứa Sau Nó Viết Quẹt Bôi Bẩn Điểm Dòm Của Mày Nhé |
| Vòng Nguy Hiểm Rẽ Bọn (dangerous structure) | Đội Hình Đẩy Căng Chằng Dây Dễ Lật Cụ Đâm Mâm Xéo Nhau (serialization cycle) |
| Khởi Tạo Vòng Sinh Trọn Vẹn (whole-transaction retry) | Đuổi Cút Chết Cuộn Sạch Bắt Tay Test Chốt Và Khởi Lại Trọng Mã SQL Đời Mới (transaction mới) |
| Chống Lún Lệnh Áo Kép (idempotent attempt) | Vẫn Đỉnh Mã Áo Đó KHÔNG Sút Lệnh Nhét Chết Bổ Sung (Không tạo business effect) |
| Rụt Giò Trọng Cuộc Cứ (bounded backoff) | Đóng Giới Cú Phọt, Thêm Nhịp Dịch Rụt Kéo Thời Lượng Limit Đo (Attempt cap, delay có jitter) |

## 7. Trạm Dịch Đường Dây (Điều hướng)

- [Soi Lỗi Lép Boundary Chết Nát](broken-code.md)
- [Bắt Bệnh Timeline SSI Chết Lúc Nào](analysis.md)
- [Lắp Mũi Trạm Tái Sinh Vững Ốp Khóa (Idempotency)](solutions.md)
- [Sân Bãi Kéo DB Trận Rạch (Testcontainers)](experiments.md)
- [Nền Móng Bọc Mù Isolation Và DB Dòm Bóng](../../concepts/isolation-levels.md)
- [Sóng Giằng Deadlock Rút Vòng Sinh Phép](../../concepts/deadlocks-and-retries.md)
- [Kéo Đo Concurrency Vực Sát](../../concepts/concurrency-testing.md)

## 8. Hố Sập Nổ Bể Chảo Trên Chiến Trường Production (Hậu quả)

- Oái Mẽ Cút Mã `40001` Chảy Trào Lọt Xuống Khách Thành Trái Đạn HTTP 500 Tuy Việc Có Thể Xong Nếu Tái Đấm;
- Nấu Trọng Trận Doomed (Giao Dịch Đã Chết) Nhét Retry Dọng Ác Hưởng Cáo `25P02`;
- Đánh Retry Lì Lợm Lên DB Sập Tịt Họng Khuếch Đại Phá (conflict amplification);
- Đọc Cứt Mũi Già Khú (stale snapshot) Đục Chết Não Oác Quyết Lệnh Cũ Chứ Hổng Thấy Người Vừa Phá Thắng Bảng;
- Nhảy Tóp Tép Gửi Tin Notification Trước Lúc Búa Đập Dội Trùng Ụp Xé Đôi Dù DB Trào Rollback;
- Mất Khách Chờ Sập Caller Ọc Phọt Cắm Lại Reservation Thứ Hai Oan Do Command Đéo Neo Đất Bền Chắc;
- Bơm Máy Lắm Nút Trạm Scale-out Chồng Chéo Siết Tụ SERIALIZABLE Hút Phá Mảng Lọng Chung 1 Predicate!

## 9. Đơn Thuốc Soi Khớp Lại (Hướng sửa khuyến nghị)

1. Cất `SERIALIZABLE` Vo Rọ CHỈ DÙNG Lúc Bãi Trượt Invariant Chằng Chéo Cực Lớn Giữa Bọn Đọc-Viết (read-write set) Cần Trói Phép Nhé. Constraint Dữ Lưới Gọn / Lách Nhẹ Update Có Predicate Nên Trọng Đất Rẻ Trụ Đẹp Trước.
2. Ép Trấn Dây Đeo Áo Trực Tiếp Thằng Đệ Chạy Bean (transactional attempt bean) Phải Ngó Ra Xét Lệnh `current_setting('transaction_isolation')`.
3. Nhét Lão Điều Hành Khởi Động Đấm Trút (Coordinator) Lên Mâm Tầng NGOÀI Cột Giao Dịch Nhen; Áp Cửa Qua Tường Proxy Nhé.
4. Xẻ Dây Soi Cáo Tử SQLSTATE `40001`, Vuốt Rơi Cuộn Lại Trắng Nát Đáy Ròi Sóng Cắp Áp MỚI RELOAD TRỌN TỪNG CHỮ (reload mọi state).
5. Vòng Tái Chiến Retry Khóa Chốt Tổng Vòng Phọt (Attempt Cap), Rụt Mỏ Dãn Tụt Trễ Nhấp Exponential Backoff + Jitter Đuôi Limit.
6. Ôm Dính Command ID; Đút Kép Bản Phán Quyết/Hộp Giấy Két Đáy Cùng Transaction Lúc Thành Công Oai Ác (successful transaction).
7. ÉO Rảnh Đi Bùn Phọt Retry Mấy Cái Lỗi Tịt Về Mõm Nghiệp Vụ (business rejection), Input Gãy Độc Hay Hộc Đạn Éo Nằm Nhóm Cấp Phép Rìa Nhé!

## 10. Lúc Nào Kéo Xuống Sân (Khi phù hợp)

Kiếm Giáp `SERIALIZABLE` Trọng Áo Lắp Xuyên Bảng Dòng (predicate/nhiều rows), Trống Bọn Xoáy Giao Nát Hướng Cùng Rạch Đồng Hàng. Đội Ngũ Bắt Mạch Xịn Sẵn Đứa Nuôi Rollback/Retry. Với Tụ Áp Chảo Cục Nóng Đu Đáy Oan Hay Thằng Óc Nặng Xử Lý Kép Tệ Thì Chuyển Trực Tiết Đệm Dòng Explicit Guard/Conditional/Tranh Đẩy Tịch Queue Ngó Lệ Trơn Sáng Nhất Nhé.

## 11. Bờ Lề Xóm Câu Chuyện (Phạm vi)

Ánh Bãi Khoanh Đo Dọc Retry Giữa Trục Thẳng Ục 40001 Này Nhen Nhóc. Đào Chén Hụt Write Skew Gốc Ảo Ở `DB-005`; Ục Vòng Khóa PostgreSQL Deadlock `40P01` Kéo Sập Rìa `DB-008`; Cuộn Lệnh Tráo Advisor Cưa Vượt Ở Giao Dịch Chết Doomed Rách Sót Kẹp Hẹp Ở `SPR-006` Nhé.
