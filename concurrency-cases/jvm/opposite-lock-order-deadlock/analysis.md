# Wait-for cycle và nguyên nhân gốc

## Trạng thái ban đầu

Lúc đầu A và B chưa ai khoá. Có hai thread: T1 đi từ A sang B; T2 đi từ B sang A.

## Interleaving tạo deadlock

Hãy xem cách chúng tự khoá chân nhau:

| Bước | T1 | T2 | Wait-for graph |
| --- | --- | --- | --- |
| 1 | Lấy Lock-A | | T1 ôm A |
| 2 | | Lấy Lock-B | T2 ôm B |
| 3 | Xin Lock-B (đợi mòn mỏi) | | T1 → T2 |
| 4 | | Xin Lock-A (cũng đợi luôn) | T1 → T2 → T1 |

Chúng ta có một cái vòng lặp vô hình (cycle). Không ông nào chịu buông tay trước để ông kia làm, thế là treo cả đôi.

> **Nói ngắn gọn:** Khăng khăng ôm cái mình có, đòi bằng được cái người kia cầm, không ai nhường ai.

## Expected và actual

| Khía cạnh | Mong đợi | Thực tế |
| --- | --- | --- |
| Progress | Xử lý lần lượt từng cái trơn tru | Kẹt cứng cả đám |
| Balance | Tổng tiền không đổi | Trạng thái chưa kịp sửa nhưng tài nguyên thì bị chiếm dụng |
| Failure | Quá thời gian thì huỷ bỏ gọn gàng | `lock()` nó không biết khái niệm deadline là gì |
| Diagnostics | Quăng lỗi để dễ fix | JVM đứng im chứ chả tự báo lỗi giúp mình |

## Nguyên nhân theo từng lớp

Có 4 yếu tố tạo ra deadlock: khóa độc quyền (mutual exclusion), giữ rồi chờ (hold-and-wait), không bị tước đoạt (no preemption), và chờ vòng tròn (circular wait). Lỗi trực tiếp ở đây là chờ vòng tròn do thứ tự lock bị ngược.
`ReentrantLock` không giúp tự gỡ vòng lặp. Spring hay Database không liên quan tới chỗ lỗi này, đây là chuyện của RAM.

## Định hướng tất định (Deterministic ordering) phá cycle

Bí quyết ở đây là: Hãy luôn sắp xếp thứ tự khoá dựa trên một ID duy nhất.
Ví dụ: Thằng nào ID bé hơn thì khoá trước. Vậy thì dù A→B hay B→A, cả T1 và T2 đều phải xin lock thằng có ID nhỏ trước. Thằng nào lấy được thì đi tiếp, thằng kia chờ. Cái vòng lẩn quẩn sẽ bị phá vỡ! 

Lưu ý: Nếu không có ID, dùng các so sánh khác mà bị bằng nhau thì phải có quy tắc phụ (tie-breaker) rõ ràng.

## Giới hạn thời gian acquire và thử lại

Bạn có thể dùng `tryLock` hay `lockInterruptibly` để không bị kẹt vô tận. Nếu không xin được lock thứ hai, nhớ phải nhả lock thứ nhất ra ở `finally`, nghỉ ngơi một chút (backoff) rồi làm lại từ đầu.
Tuyệt đối không update tiền bạc hay gọi ra ngoài khi chưa xin đủ 2 lock.

## Lỗi, gián đoạn và sự cố hệ thống

- Đang chờ mà bị ngắt? Chuyển ngay cái `InterruptedException` thành lỗi nghiệp vụ.
- Nguyên tắc vàng: Chỉ nhả (unlock) những lock mình đang cầm và nhả theo thứ tự ngược lại lúc lấy.
- Kiểm tra tính hợp lệ dữ liệu (validation) sau khi gom đủ đồ (lock).
- Có văng Exception gì thì `finally` cũng phải lo dọn dẹp nhả lock.
- App sập thì deadlock hết, nhưng dữ liệu trên RAM cũng bay màu.

## Multi-instance và biên giới PostgreSQL

Cách sửa lock ở RAM này chỉ xài cho 1 instance thôi. Ở PostgreSQL, DB nó có thuật toán phát hiện và giải quyết riêng (rollback 1 transaction làm nạn nhân). 

## Hậu quả

- Request đứng im, thread pool cạn sạch;
- Các connection bị ngâm giấm;
- Upstream thấy lỗi cứ ráng gửi thêm request lại (retry storm);
- Health check thì báo xanh nhưng hệ thống thì đang hấp hối;
- Restart thì hết treo nhưng lại không chữa tận gốc.

## Quan sát

Khi bị, hãy chụp thread dump. Dùng hàm `ThreadMXBean.findDeadlockedThreads` để mò ra mấy thread cứng đầu. Nhớ ghi log để theo dõi.
