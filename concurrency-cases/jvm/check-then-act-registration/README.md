# Bài toán JVM-002 — Đăng Ký Tài Nguyên Theo Mô Hình "Kiểm Tra Rồi Hành Động" (Check-Then-Act)

## 1. Tóm tắt vấn đề (Overview)

Hãy tưởng tượng bạn có một bộ đăng ký (registry) lưu trên RAM để quản lý các đối tượng `ManagedResource` theo `resourceKey`. Nhiều anh em lầm tưởng cứ xài `ConcurrentHashMap` là auto an toàn trong môi trường đa luồng (thread-safe). Nhưng thực tế thì không phải vậy, lỗi nằm ở việc bạn chia quy trình ra làm hai bước rời rạc:

```text
Kiểm tra key chưa tồn tại → Khởi tạo và đưa tài nguyên mới vào Map
```

Khi code chạy thật, hai luồng (thread) khác nhau hoàn toàn có thể cùng lúc thấy key chưa tồn tại, thế là cả hai thi nhau tạo mới tài nguyên. Dù cuối cùng cái Map của bạn chỉ lưu một đối tượng, nhưng đối tượng kia bị bỏ rơi, không ai xài đến mà nó vẫn chiếm dụng tài nguyên hệ thống (Thread, Socket, v.v.) một cách lãng phí.

Để bài toán này chạy chuẩn xác trên JVM, mình cần đảm bảo các **Quy tắc Bất biến (Business Invariants)** sau:

```text
- Mỗi resourceKey chỉ được phép liên kết tối đa với một đối tượng ManagedResource đang hoạt động.
- Mọi luồng yêu cầu cấp phát (caller) cho cùng một key phải nhận về chính xác cùng một phiên bản tài nguyên do registry quản trị.
- Hàm kiến tạo (Factory) chỉ được cấp phép thực thi duy nhất một lần cho quá trình đăng ký một key.
```

> **Nguyên tắc kỹ thuật:** Cấu trúc Map của bạn có thread-safe đi nữa thì không có nghĩa là nguyên cả một đoạn code logic (check rồi mới act) của bạn cũng thread-safe. Cái khe hở giữa lúc "kiểm tra" và "hành động" là đủ để luồng khác nhảy vào phá hỏng trạng thái rồi.

## 2. Các Thuật ngữ Chuyên ngành (Terminology)

| Thuật ngữ | Hiểu đơn giản là gì? |
| --- | --- |
| Mô hình `check-then-act` | Bạn check điều kiện trước, ok rồi mới làm. Nhưng giữa lúc check xong và lúc làm thì có luồng khác xía vào. |
| Thao tác phức hợp (`compound action`) | Là một cụm nhiều hành động gộp lại. |
| Thao tác Map nguyên tử (`atomic map operation`) | Là cái thao tác mà bạn can thiệp vào Map dứt khoát 1 lần, luồng khác không có cơ hội chèn ngang. |
| Phương thức `computeIfAbsent` | Hàm này giúp bạn làm đúng 1 việc: Nếu key chưa có thì tạo mới và bỏ vào Map luôn (bao nguyên tử). |
| Điểm hiệu lực duy nhất (`linearization point`) | Là cái khoảnh khắc chốt hạ mà cả hệ thống đều công nhận hành động đã hoàn tất. |
| Tài nguyên vô thừa nhận (`orphan resource`) | Hàng đã tạo ra nhưng bị lãng quên, không ai gom rác (thu hồi). |
| Công bố an toàn (`safe publication`) | Share cho các luồng khác một cái đối tượng khi nó đã được tạo ra hoàn chỉnh, ngon nghẻ rồi. |

## 3. Bối cảnh nghiệp vụ (Business Context)

Ví dụ ứng dụng của bạn cần tạo một Client riêng cho mỗi khách hàng (Tenant):

- Luồng T1 đòi cấp tài nguyên cho `tenant-a`.
- Luồng T2 cũng đòi đúng cái đó luôn, cùng lúc.
- Hàm `ManagedResourceFactory.open(...)` sẽ mở ra các thứ rất nặng nề như Thread, Socket.
- Registry có nhiệm vụ lấy đồ đã có ra xài lại chứ không được sinh ra rác.

Lưu ý là bài toán này mình chỉ nói ở phạm vi RAM của 1 server thôi nhé (Local Registry). Còn nếu bạn cần quản lý độc quyền trên nhiều server (Cluster) thì phải xem bài `DB-006` và `DIST-001`.

## 4. Các Thực thể và Trạng thái chia sẻ (Shared state & Contention)

| Thành phần | Vai trò |
| --- | --- |
| Tài nguyên chia sẻ | Thằng `ManagedResourceRegistry` (Singleton). |
| Bộ nhớ trạng thái | Cái map `ConcurrentMap<String, ManagedResource>`. |
| Chủ thể cạnh tranh | Mấy cái Request/Worker thread đang tranh nhau chạy. |
| Chuỗi thao tác lỗi | `get(key) → open(key) → put(key, resource)` |
| Vị trí đứt gãy | Cái chỗ hở giữa lúc `get` xong nhưng chưa kịp `put`. |
| Ranh giới Giao dịch | Chạy trong RAM, chẳng liên quan gì đến Database Transaction cả. |
| Phạm vi Bất biến | Chỉ tính trong 1 cục Application Instance (1 server). |

## 5. Điều hướng Tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế Hệ Thống Đặt Chỗ (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Luồng Tương Tranh (analysis.md)](analysis.md)
- [Giải Pháp Cấu Hình Phù Hợp (solutions.md)](solutions.md)
- [Thực Nghiệm Khóa Bi Quan Với Testcontainers (experiments.md)](experiments.md)
- [Tổng Quan Về Mô Hình Bộ Nhớ Java (Java Memory Model)](../../concepts/java-memory-model-and-atomicity.md)
- [Phương Pháp Kiểm Thử Đồng Thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## 6. Tác Động Tới Hệ Thống (Production Impact)

### Hệ Quả Kỹ Thuật
- Tạo tài nguyên mới liên tục một cách lãng phí cho cùng 1 key.
- Mấy luồng chậm chân tạo xong không được ghi nhận vào Map.
- Gây rò rỉ Thread, Socket nặng nề.
- Các luồng gọi chung 1 key nhưng lại cầm về mấy cái object vật lý khác nhau.
- Lúc tắt hệ thống dọn dẹp thì bó tay, vì chỉ dọn được cái nào có trong Map.

### Hệ Quả Nghiệp Vụ
- Hệ thống vô tình mở nhiều kết nối cho cùng một ông khách hàng.
- Bắn event trùng lặp, hoặc làm nghẽn mẹ nó giới hạn (rate limit) của đối tác.
- Tải cao là chết ngắc vì cạn kiệt Connection hay File Descriptor.
- Lỗi lúc bị lúc không (do tuỳ thời điểm luồng nó chạy thế nào), nên fix bug cực kỳ đau đầu.

## 7. Khuyến Nghị Áp Dụng (Best Practices)

- Hãy xài `computeIfAbsent` nếu quá trình tạo mới nó nhẹ, không thao tác lại vào chính cái Map và không đụng tới I/O mạng mẽo quá lâu.
- Nếu tạo tài nguyên rất nặng hoặc có side-effect, hãy dùng chiêu Điền Trống (Placeholder) kiểu `FutureTask`. Nghĩa là thằng nào nhanh chân thì đi tạo thật, các thằng khác đứng chờ đồng bộ lấy kết quả thôi.
- Cùng lắm nếu muốn dùng `putIfAbsent` (chấp nhận tạo thừa), thì NHẤT ĐỊNH phải dọn dẹp (clean up) mấy cái đối tượng thừa thãi bị vứt đi.
- Đừng bao giờ lôi cái Local Registry này ra để giải quyết bài toán độc quyền cho kiến trúc nhiều server (Multi-instance).

## 8. Phân Tích Phương Án Chọn Lựa (Architectural Alternatives)

- **Sử dụng `computeIfAbsent`**: Ngon, hợp cho local cache, dễ dùng, ít lỗi.
- **Mô hình Placeholder (`FutureTask`)**: Cực xịn nếu cần khởi tạo nặng, buộc phải share đồng bộ. Thêm nữa còn dễ clean up hoặc làm cơ chế Retry.
- **Hợp nhất `putIfAbsent` & Cleanup**: OK nếu bạn lười, chấp nhận tạo rác tạm thời nhưng nhớ phải dọn dẹp được nó.
- **Từ khóa `synchronized`**: Xài tạm nếu registry cực nhỏ, ít xài, chấp nhận việc tất cả các luồng phải xếp hàng chờ đợi nhau.
- **Ràng buộc Database hoặc Điều phối Phân tán**: Dùng khi bạn chơi hệ đa máy chủ, hoặc cần lưu trữ giữ liệu kể cả khi sập server khởi động lại.
