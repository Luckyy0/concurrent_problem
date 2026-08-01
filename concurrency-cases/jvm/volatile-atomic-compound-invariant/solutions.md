# Giải pháp, code đã sửa và các đánh đổi

## Giải pháp 1: CAS trên một BudgetState bất biến (immutable)

Gom hai counter vào một object bất biến và chỉ công bố state bằng CAS trên một
`AtomicReference`.

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

            BudgetState next = new BudgetState(
                    current.active(),
                    current.pending() + 1
            );
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

### Vì sao invariant được bảo vệ

- `BudgetState` chứa một snapshot nhất quán của cả hai counter.
- Bước kiểm tra limit và quá trình reserve dùng chung một state; quá trình CAS thành công là linearization point.
- Transition pending-to-active được công bố bằng một quá trình hoán đổi tham chiếu (reference swap) và bảo toàn `used`.
- Thread thua cuộc (loser) trong CAS sẽ đọc lại state mới nhất trước khi đưa ra quyết định.
- `view()` đọc reference một lần, không ghép các counter từ hai thời điểm khác nhau.
- Tình trạng underflow bị từ chối thay vì âm thầm tạo ra capacity giả.

> **Nói ngắn gọn:** muốn atomicity cho một quy tắc gồm nhiều field, hãy thực hiện CAS một state
> chứa đủ các field đó, không CAS từng mảnh riêng lẻ.

### Điều kiện của CAS loop

`operation.apply(current)` có thể chạy lại nhiều lần nếu CAS thất bại, nên lambda phải
không có side-effect. Việc log “connection activated”, phát hành sự kiện (publish event) hoặc gọi remote API
chỉ được thực hiện sau khi transition thành công, ở bên ngoài hàm có khả năng retry (retryable function).

CAS bảo vệ invariant tổng hợp (aggregate invariant) nhưng không xác nhận định danh (identity) của callback. Nếu
một callback xử lý trường hợp reservation thất bại chạy hai lần trong khi một reservation khác vẫn đang
pending, lần chạy thứ hai có thể lấy nhầm slot của reservation kia. Khi callback có
thể duplicate/out-of-order, cần một permit handle hoặc state machine cho reservation.

## Giải pháp 2: một ReentrantLock bảo vệ toàn bộ state

Lock là lựa chọn rõ ràng khi quá trình transition phức tạp hoặc cần tính công bằng (fairness). Dùng plain
`int`; không cần AtomicInteger ở bên trong cùng một lock.

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

`connectionClosed()` phải dùng chung một lock và kiểm tra `active > 0`; đoạn method đó
được lược đi vì giống `creationFailed()`. Remote handshake luôn chạy ở bên ngoài lock.
Không giữ lock trong quá trình xử lý network I/O.

Nếu cần thời gian chờ có giới hạn (bounded wait), hãy dùng `tryLock(timeout, unit)`, xử lý
`InterruptedException`, khôi phục trạng thái ngắt (interrupt status) và định nghĩa rõ việc lock timeout
là một sự từ chối hay là lỗi kỹ thuật.

## Giải pháp 3: một AtomicInteger cho tổng số used slots

Nếu tính đúng đắn (correctness) chỉ phụ thuộc vào tổng số lượng slot, hãy giảm bớt phần state cần synchronize:

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

Quá trình tạo connection giữ lại slot từ lúc pending đến lúc active; việc transition không
đụng chạm tới counter. Các số liệu (metric) Active/pending có thể được theo dõi riêng như một khả năng quan sát (observability)
xấp xỉ, nhưng không được đưa ngược vào để quyết định việc phân bổ capacity.

## Giải pháp 4: Semaphore và permit handle

`Semaphore` mô hình hóa capacity một cách trực tiếp. Một permit được giữ xuyên suốt quá trình pending và
active, rồi được release đúng một lần thông qua một handle:

```java
public final class ConnectionPermit implements AutoCloseable {

    private final Semaphore semaphore;
    private final AtomicBoolean released = new AtomicBoolean();

    ConnectionPermit(Semaphore semaphore) {
        this.semaphore = semaphore;
    }

    @Override
    public void close() {
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

Quá trình tạo connection thất bại sẽ gọi `permit.close()`. Khi quá trình tạo thành công, connection object
sở hữu permit đó và tự đóng nó lại trong vòng đời (lifecycle) `close()`. `AtomicBoolean` làm cho quá trình duplicate
close trở thành no-op và ngăn chặn việc over-release.

Semaphore giải quyết bài toán về tổng capacity, không tự cung cấp snapshot phân tách active/pending.
Một fair semaphore có thể giảm tình trạng đói tài nguyên (starvation) nhưng thường làm giảm thông lượng (throughput); chỉ nên bật khi
tính công bằng (fairness) là một yêu cầu (requirement) đã được đo lường cẩn thận.

## So sánh các đánh đổi

| Phương án | Invariant | Thread thua cuộc (Loser) | Contention/fairness | Khả năng quan sát (Observability) | Multi-instance |
| --- | --- | --- | --- | --- | --- |
| `AtomicReference<BudgetState>` | Chính xác trên nhiều counter | CAS retry rồi reject nếu đã đầy (full) | Lock-free, không đảm bảo fairness | Snapshot active/pending chính xác | Chỉ một JVM |
| `ReentrantLock` | Chính xác trên nhiều counter/transition | Bị block hoặc timeout | Có thể cấu hình fairness | Có snapshot nếu lấy dưới lock | Chỉ một JVM |
| Một `AtomicInteger used` | Chính xác trên tổng lượng slot | CAS retry rồi reject | Đơn giản, không đảm bảo fairness | Việc chia tách chi tiết (breakdown) chỉ nên là metric phụ | Chỉ một JVM |
| `Semaphore` + handle | Chính xác theo permit lifecycle | `tryAcquire` fail hoặc chờ với một bounded wait | Fair/non-fair tùy theo cấu hình | Chỉ biết lượng permit khả dụng, không có breakdown | Chỉ một JVM |
| Hai `AtomicInteger` độc lập | Không bảo vệ được compound invariant | Có thể có nhiều actor cùng thắng | Nhanh nhưng sai | Snapshot không có tính nhất quán | Chỉ một JVM |

Không có thông số cụ thể chung nào cho throughput/latency. Việc lựa chọn nên dựa trên contention thực tế,
mức độ phức tạp của transition, tính fairness và mức quan trọng của exact breakdown.

## Các chính sách về Failure, retry và lifecycle

- Giữ chỗ (reserve) trước khi thực hiện handshake; nếu không còn slot, hãy fail-fast hoặc đưa vào queue có giới hạn.
- Handshake phải có thời gian timeout; trường hợp thất bại (failure) phải thực hiện release pending/permit đúng một lần.
- Trường hợp thành công (success) sẽ chuyển quyền sở hữu (ownership) của reservation đó sang cho connection.
- Quá trình đóng (close) phải idempotent; các callback trùng lặp không được release slot thuộc về vòng đời của reservation khác.
- Việc CAS retry chỉ được phép bao quanh các quá trình transition cục bộ; tuyệt đối không đặt remote call vào bên trong một vòng lặp CAS loop.
- Nếu cập nhật state thành công nhưng quá trình phát hành sự kiện (publish event) thất bại, hãy retry event đó một cách độc lập;
  không được chạy lại transition một cách mù quáng.

## Khi nào nên dùng

- Chọn `AtomicReference` khi yêu cầu cần có exact active/pending snapshot.
- Chọn lock khi code cần ưu tiên mức độ dễ chứng minh, có nhiều transition phức tạp hoặc cần tính năng condition.
- Chọn single counter/semaphore khi capacity permit mới chính là invariant cốt lõi.
- Thêm một định danh cho reservation (reservation identity) khi các callback có thể bị retry hoặc out-of-order.
- Chuyển sang sự điều phối tập trung bên ngoài (external coordination) nếu quota được dùng chung giữa nhiều node.

## Lưu ý khi áp dụng thực tế

- Số liệu (Metric): `active`, `pending`, `used`, `limit`, số lượng bị từ chối, số lần retry CAS,
  số lượng timeout/failure trong quá trình handshake và các lần đóng trùng lặp (duplicate close).
- Tạo cảnh báo (alert) khi `used > limit` hoặc số counter bị âm; đây là những vi phạm invariant, không chỉ
  đơn giản là metric bất thường.
- Ghi lại nhật ký (log) các quá trình transition thất bại kèm theo mã định danh vòng đời/reservation, không log các thông tin xác thực (credential).
- Endpoint health check phải đọc từ một state snapshot, không được gọi liên tiếp nhiều getter độc lập.
- Giới hạn thời gian (duration) pending; những reservation bị treo quá lâu phải bị timeout và dọn dẹp.
- Không tự động “sửa” counter bằng cách set lại về mức limit; cần phải điều tra vấn đề về rò rỉ vòng đời (lifecycle leak) và
  đối soát lại với connection owner.
- Khi diễn ra shutdown, ngừng nhận các reservation mới rồi tiến hành đóng có kiểm soát các active connection; state cục bộ không cần tính bền vững để rollback (durable rollback).
