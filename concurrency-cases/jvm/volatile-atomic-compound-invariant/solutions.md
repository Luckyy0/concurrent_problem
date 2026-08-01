# Giải pháp, code đã sửa và các đánh đổi

## Giải pháp 1: CAS trên một BudgetState bất biến (immutable)

Chiêu này cực hay: gom cả hai biến đếm vào chung một cái object bất biến (không thể sửa đổi sau khi tạo). Xong rồi chỉ dùng `AtomicReference` để cập nhật cả cục này thông qua CAS.

```java
package com.example.connection;

public record BudgetState(int active, int pending) {

    public BudgetState {
        if (active < 0 || pending < 0) {
            throw new IllegalArgumentException("budget counters must be non-negative");
        }
    }

    public int used() {
        return Math.addExact(active, pending);
    }
}
```

```java
package com.example.connection;

import org.springframework.stereotype.Component;

import java.util.concurrent.atomic.AtomicReference;
import java.util.function.UnaryOperator;

@Component
public class ProviderConnectionBudget {

    private final int limit;
    private final AtomicReference<BudgetState> state =
            new AtomicReference<>(new BudgetState(0, 0));

    public ProviderConnectionBudget(ConnectionBudgetProperties properties) {
        if (properties.maxConnections() <= 0) {
            throw new IllegalArgumentException("maxConnections must be positive");
        }
        this.limit = properties.maxConnections();
    }

    public boolean tryReserveCreation() {
        while (true) {
            BudgetState current = state.get();
            if (current.used() >= limit) {
                return false;
            }

            // Tạo state mới dựa trên state cũ
            BudgetState next = new BudgetState(
                    current.active(),
                    current.pending() + 1
                );
            // Thử chốt đơn, ai nhanh tay thì thắng
            if (state.compareAndSet(current, next)) {
                return true;
            }
        }
    }

    public void creationSucceeded() {
        transition(current -> {
            requirePositive(current.pending(), "no pending creation");
            return new BudgetState(
                    current.active() + 1,
                    current.pending() - 1
            );
        });
    }

    public void creationFailed() {
        transition(current -> {
            requirePositive(current.pending(), "no pending creation");
            return new BudgetState(
                    current.active(),
                    current.pending() - 1
            );
        });
    }

    public void connectionClosed() {
        transition(current -> {
            requirePositive(current.active(), "no active connection");
            return new BudgetState(
                    current.active() - 1,
                    current.pending()
            );
        });
    }

    public BudgetView view() {
        BudgetState snapshot = state.get();
        return new BudgetView(snapshot.active(), snapshot.pending(), limit);
    }

    BudgetState stateForTest() {
        return state.get();
    }

    private BudgetState transition(UnaryOperator<BudgetState> operation) {
        while (true) {
            BudgetState current = state.get();
            BudgetState next = operation.apply(current);
            if (next.used() > limit) {
                throw new IllegalStateException("budget invariant violated");
            }
            if (state.compareAndSet(current, next)) {
                return next;
            }
        }
    }

    private static void requirePositive(int value, String message) {
        if (value <= 0) {
            throw new IllegalStateException(message);
        }
    }
}
```

### Vì sao cách này lại xịn?

- `BudgetState` luôn giữ một bức ảnh (snapshot) cực kỳ chính xác và đồng bộ của cả 2 biến đếm.
- Thao tác check limit và giữ chỗ diễn ra cùng lúc trên một trạng thái duy nhất. Cú CAS ăn điểm chính là "khoảnh khắc quyết định" (linearization point).
- Chuyển từ pending sang active diễn ra cái vèo qua một lần tráo đổi (swap), tổng số vé luôn được bảo toàn nguyên vẹn.
- Nếu thread nào thao tác trễ nhịp (thua CAS), nó tự động đọc lại số mới nhất rồi tính tiếp.
- `view()` chỉ lấy snapshot đúng 1 lần, không còn vụ ghép 2 con số từ 2 thời điểm khác nhau.
- Không sợ biến đếm bị âm lén lút, vì sai là nó báo lỗi (throw exception) liền.

> **Nói ngắn gọn:** Muốn an toàn cho bộ quy tắc gồm nhiều biến, cứ bỏ chung tụi nó vào 1 gói rồi dùng CAS cả gói. Đừng có táy máy update lẻ tẻ từng món.

### Điều kiện của vòng lặp CAS

Hàm `operation.apply(current)` có thể bị gọi đi gọi lại mấy vòng liền nếu bị tranh giành. Thế nên, cấm tuyệt đối việc bỏ code có "tác dụng phụ" (side-effect) vào đây. Viết log, gửi sự kiện (event), hay gọi API ngoài thì chờ hàm transition chạy xong xuôi, cầm chắc phần thắng rồi hẵng làm nhé.

Nhược điểm: CAS ngon nhưng nó vẫn không nhận diện được từng giao dịch (identity). Nếu một cái callback xui xẻo chạy đúp 2 lần, nó vẫn có thể trừ nhầm slot của anh khác. Muốn trị dứt điểm cái này phải dùng thẻ cấp phép (permit handle) hoặc state machine đính kèm ID.

## Giải pháp 2: Một cái ổ khóa (ReentrantLock) bảo vệ tất cả

Lock là giải pháp cực kỳ dễ thở khi nghiệp vụ đổi trạng thái rối rắm hoặc bạn cần phân định rõ ràng trước sau (fairness). Đã dùng lock rồi thì mấy biến bên trong xài `int` thường là đủ, vứt `AtomicInteger` đi cho rảnh.

```java
@Component
public class LockedProviderConnectionBudget {

    private final int limit;
    private final ReentrantLock lock = new ReentrantLock();
    private int active;
    private int pending;

    public LockedProviderConnectionBudget(ConnectionBudgetProperties properties) {
        if (properties.maxConnections() <= 0) {
            throw new IllegalArgumentException("maxConnections must be positive");
        }
        this.limit = properties.maxConnections();
    }

    public boolean tryReserveCreation() {
        lock.lock();
        try {
            if (active + pending >= limit) {
                return false;
            }
            pending++;
            return true;
        } finally {
            lock.unlock();
        }
    }

    public void creationSucceeded() {
        lock.lock();
        try {
            if (pending <= 0) {
                throw new IllegalStateException("no pending creation");
            }
            pending--;
            active++;
        } finally {
            lock.unlock();
        }
    }

    public void creationFailed() {
        lock.lock();
        try {
            if (pending <= 0) {
                throw new IllegalStateException("no pending creation");
            }
            pending--;
        } finally {
            lock.unlock();
        }
    }

    public BudgetView view() {
        lock.lock();
        try {
            return new BudgetView(active, pending, limit);
        } finally {
            lock.unlock();
        }
    }
}
```

Hàm `connectionClosed()` thì tương tự như `creationFailed()` thôi, mình cắt đi cho gọn. Chú ý: Kết nối mạng, gọi API từ xa thì **tuyệt đối phải làm bên ngoài khối lock** nhé. Khóa rồi mới gọi API, mạng nó rớt cái là cả đám chết đứng chôn chân chờ nhau.

Nếu muốn đợi một chút rồi thôi, xài `tryLock(timeout, unit)` và nhớ xử lý ngắt (interrupt) tử tế.

## Giải pháp 3: Một AtomicInteger quản lý mọi slot

Nếu cuối cùng bạn cũng ngộ ra rằng cái bạn quan tâm duy nhất là "Tổng vé là bao nhiêu?", thì hãy dẹp bớt sự rườm rà đi:

```java
public final class AtomicSlotBudget {

    private final int limit;
    private final AtomicInteger used = new AtomicInteger();

    public AtomicSlotBudget(int limit) {
        if (limit <= 0) {
            throw new IllegalArgumentException("limit must be positive");
        }
        this.limit = limit;
    }

    public boolean tryAcquire() {
        while (true) {
            int current = used.get();
            if (current >= limit) {
                return false;
            }
            if (used.compareAndSet(current, current + 1)) {
                return true;
            }
        }
    }

    public void release() {
        while (true) {
            int current = used.get();
            if (current <= 0) {
                throw new IllegalStateException("no slot to release");
            }
            if (used.compareAndSet(current, current - 1)) {
                return;
            }
        }
    }
}
```

Từ pending sang active thì cũng kệ tía nó, tổng vẫn y chang nên mình chả cần đụng vô counter làm gì. Muốn vẽ vời active/pending thì đếm riêng chỉ để hóng chuyện (observability) thôi, đừng lấy nó ra làm luật.

## Giải pháp 4: Chơi hẳn Semaphore và vé thông hành (permit handle)

`Semaphore` thì đúng sinh ra để mô phỏng "sức chứa" rồi. Lấy 1 vé, giữ đó xài từ lúc pending cho tới active, xong xuôi thì nhả ra (release) đúng 1 lần:

```java
public final class ConnectionPermit implements AutoCloseable {

    private final Semaphore semaphore;
    private final AtomicBoolean released = new AtomicBoolean();

    ConnectionPermit(Semaphore semaphore) {
        this.semaphore = semaphore;
    }

    @Override
    public void close() {
        // Có thả vé ngàn lần thì cũng chỉ nhả 1 lần thôi nha cưng
        if (released.compareAndSet(false, true)) {
            semaphore.release();
        }
    }
}
```

```java
public final class SemaphoreConnectionBudget {

    private final Semaphore semaphore;

    public SemaphoreConnectionBudget(int limit) {
        if (limit <= 0) {
            throw new IllegalArgumentException("limit must be positive");
        }
        this.semaphore = new Semaphore(limit);
    }

    public Optional<ConnectionPermit> tryAcquire() {
        if (!semaphore.tryAcquire()) {
            return Optional.empty();
        }
        return Optional.of(new ConnectionPermit(semaphore));
    }
}
```

Với trò dùng `AtomicBoolean` ở trên, ai có trót tay gọi close 2 lần thì hệ thống vẫn tỉnh rụi (no-op), đảm bảo không nhả thừa vé.

Semaphore tập trung vào tổng thể, nó không phân loại rành rọt pending hay active. Bạn có thể bật chế độ công bằng (fair semaphore) để tránh mấy thanh niên chen lấn, nhưng đánh đổi lại thì hiệu suất (throughput) sẽ rớt thê thảm. Chú ý nhé.

## So sánh các đánh đổi

| Phương án | Bảo vệ quy tắc (Invariant) | Kẻ thua vòng lặp (Loser) | Tranh chấp/Công bằng | Độ hóng chuyện (Observability) | Dùng nhiều máy |
| --- | --- | --- | --- | --- | --- |
| `AtomicReference<BudgetState>` | Cực chuẩn xác cho nhiều biến | Chạy lại CAS rồi báo Full | Chớp nhoáng, không rảnh xếp hàng | Xem active/pending cực chuẩn | 1 máy JVM |
| `ReentrantLock` | Chuẩn đét, bao trọn gói | Bị chặn hoặc chờ đợi mỏi mòn | Có thể setup bắt xếp hàng | Dữ liệu đúng nếu lấy trong lock | 1 máy JVM |
| Một `AtomicInteger used` | Chuẩn đét phần tổng vé | Chạy lại CAS rồi báo Full | Đơn giản, không xếp hàng | Chia active/pending chỉ mang tính minh họa | 1 máy JVM |
| `Semaphore` + vé thông hành | Chuẩn theo chu kỳ vé | Fail hoặc ngồi chờ có hạn | Bật/tắt xếp hàng tùy ý | Chỉ biết còn nhiêu vé, chả biết trạng thái | 1 máy JVM |
| Dùng 2 cái `AtomicInteger` (Code lỗi) | Thua, vứt vô sọt rác | Vài thằng cùng nghĩ mình thắng | Nhanh đấy nhưng sai lè | Số liệu lộn xộn, ghép nối bậy bạ | 1 máy JVM |

Cứ đo đạc tình hình thực tế rồi hẵng chốt hạ. Đừng cố đấm ăn xôi theo lý thuyết.

## Các chính sách về Lỗi (Failure), thử lại (retry) và vòng đời (lifecycle)

- Xin số (reserve) xong xuôi mới đi bắt tay. Không có số thì báo lỗi liền hoặc cho vào hàng đợi xíu xiu.
- Bắt tay (Handshake) nhớ phải có timeout; lỡ đứt gánh thì nhả vé pending ra cho người khác xài.
- Nếu thành công, sang tên vé đó cho cái kết nối chính thức xài.
- Lúc đóng cửa (close) thì gọi mấy lần cũng kệ nó; đừng có kiểu gọi trùng mà đi nhả nhầm vé của thằng khác.
- Lặp CAS thì chỉ ôm mấy thứ chạy lẹ trong máy; còn gọi API đi xa thì vứt ra ngoài lặp, cấm cãi.
- Nếu update state ngon lành mà lỡ bước báo cáo sự kiện (event) bị xịt, thì lo đi retry cái event đó thôi. Đừng khùng mà rã nguyên khối state ra chạy lại.

## Khi nào nên dùng chiêu gì

- Xài `AtomicReference` nếu sếp bắt phải coi rành rọt active với pending mọi lúc mọi nơi.
- Xài lock (ReentrantLock) khi quy tắc đổi chác nhức đầu quá, hoặc phải đảm bảo công bằng. Code dễ bảo trì hơn vạn lần.
- Xài biến đếm tổng (single counter) hoặc semaphore khi thứ cốt lõi nhất chỉ là "nhà còn bao nhiêu chỗ?".
- Ném thêm cái ID vào nếu sợ mấy cái callback chạy loăng quăng không đúng thứ tự.
- Qua nhà hàng xóm (Redis, ZooKeeper...) nếu cục quota đó cần xài chung cho chục cái server.

## Lưu ý khi ra trận (áp dụng thực tế)

- Sắm đủ bộ đo lường: `active`, `pending`, `used`, limit, tỷ lệ chối từ, số lần retry CAS, số lần timeout.
- Cài còi báo động (alert) liền nếu `used > limit` hoặc số tự dưng âm (dấu hiệu của cái bug bự chứ hông phải chuyện giỡn).
- Ghi log đàng hoàng mấy cú sập kèm theo ID giao dịch, nhưng nhớ che giấu token đồ đạc (credential) lại nhé.
- Cái hàm check health chỉ nên chớp nhoáng chụp một tấm ảnh (snapshot) là ra về, đừng rảnh mà ngồi đọc lẻ tẻ từng con số.
- Cái nào pending quá hạn thì mạnh tay dọn dẹp (cleanup), đừng để nó chiếm dụng vô duyên.
- Không có chơi trò "hacker lỏ" tự reset counter về ban đầu nha, hãy truy lùng thằng nào đang leak vòng đời (lifecycle leak).
- Khi server tắt máy đi ngủ (shutdown), khóa cửa không nhận khách (reserve) nữa rồi cho bà con đang active từ từ rời sân. Khỏi cần tốn công lưu trữ (durable rollback) làm chi.
