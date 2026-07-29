# Giải pháp, code đã sửa và các đánh đổi

## Giải pháp 1: CAS trên một immutable BudgetState

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

- `BudgetState` chứa snapshot nhất quán của cả hai counter.
- Check limit và reserve dùng cùng state; CAS success là linearization point.
- Pending-to-active được công bố bằng một reference swap và bảo toàn `used`.
- CAS loser đọc lại state mới nhất trước khi quyết định.
- `view()` đọc reference một lần, không ghép counter từ hai thời điểm.
- Underflow bị từ chối thay vì âm thầm tạo capacity giả.

> **Nói ngắn gọn:** muốn atomicity cho một quy tắc nhiều field, hãy CAS một state
> chứa đủ các field đó, không CAS từng mảnh riêng.

### Điều kiện của CAS loop

`operation.apply(current)` có thể chạy lại nếu CAS fail, nên lambda phải
side-effect-free. Log “connection activated”, publish event hoặc gọi remote API
chỉ được thực hiện sau khi transition thành công, ngoài retryable function.

CAS bảo vệ aggregate invariant nhưng không xác nhận identity của callback. Nếu
một reservation fail callback chạy hai lần trong khi reservation khác vẫn
pending, lần thứ hai có thể lấy nhầm slot của reservation khác. Khi callback có
thể duplicate/out-of-order, cần permit handle hoặc reservation state machine.

## Giải pháp 2: một ReentrantLock bảo vệ toàn bộ state

Lock là lựa chọn rõ ràng khi transition phức tạp hoặc cần fairness. Dùng plain
`int`; không cần AtomicInteger bên trong cùng lock.

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

`connectionClosed()` phải dùng cùng lock và kiểm tra `active > 0`; đoạn method đó
được lược vì giống `creationFailed()`. Remote handshake luôn chạy ngoài lock.
Không giữ lock trong network I/O.

Nếu cần bounded wait, dùng `tryLock(timeout, unit)`, xử lý
`InterruptedException`, khôi phục interrupt status và định nghĩa rõ lock timeout
là rejection hay technical failure.

## Giải pháp 3: một AtomicInteger cho tổng used slots

Nếu correctness chỉ phụ thuộc tổng slot, giảm state cần synchronize:

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

Connection creation giữ slot từ lúc pending đến lúc active; transition không
đụng counter. Active/pending metrics có thể được theo dõi riêng như observability
xấp xỉ, nhưng không được đưa ngược vào capacity decision.

## Giải pháp 4: Semaphore và permit handle

`Semaphore` mô hình hóa capacity trực tiếp. Permit được giữ xuyên suốt pending và
active, rồi release đúng một lần bằng handle:

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

Creation failure gọi `permit.close()`. Khi creation thành công, connection object
sở hữu permit và đóng nó trong lifecycle `close()`. `AtomicBoolean` làm duplicate
close thành no-op và ngăn over-release.

Semaphore giải quyết tổng capacity, không tự cung cấp snapshot active/pending.
Fair semaphore có thể giảm starvation nhưng thường giảm throughput; chỉ bật khi
fairness là requirement đã được đo.

## So sánh các đánh đổi

| Phương án | Invariant | Loser | Contention/fairness | Observability | Multi-instance |
| --- | --- | --- | --- | --- | --- |
| `AtomicReference<BudgetState>` | Chính xác nhiều counter | CAS retry rồi reject nếu full | Lock-free, không fairness | Snapshot active/pending chính xác | Chỉ một JVM |
| `ReentrantLock` | Chính xác nhiều counter/transition | Block hoặc timeout | Có thể cấu hình fairness | Snapshot dưới lock | Chỉ một JVM |
| Một `AtomicInteger used` | Chính xác tổng slot | CAS retry rồi reject | Đơn giản, không fairness | Breakdown chỉ nên là metric phụ | Chỉ một JVM |
| `Semaphore` + handle | Chính xác permit lifecycle | `tryAcquire` fail hoặc bounded wait | Fair/non-fair tùy cấu hình | Available permits, không breakdown | Chỉ một JVM |
| Hai `AtomicInteger` độc lập | Không bảo vệ compound invariant | Có thể nhiều actor cùng thắng | Nhanh nhưng sai | Snapshot không nhất quán | Chỉ một JVM |

Không có throughput/latency number chung. Chọn dựa trên contention thực tế,
transition complexity, fairness và mức quan trọng của exact breakdown.

## Failure, retry và lifecycle policy

- Reserve trước handshake; nếu không có slot, fail-fast hoặc queue có giới hạn.
- Handshake phải có timeout; failure release pending/permit đúng một lần.
- Success chuyển ownership của reservation sang connection.
- Close idempotent; callback duplicate không được release slot của lifecycle khác.
- CAS retry chỉ bao quanh local transition; không đặt remote call trong CAS loop.
- Nếu update state thành công nhưng publish event thất bại, retry event độc lập;
  không chạy lại transition mù quáng.

## Khi nào nên dùng

- Chọn `AtomicReference` khi exact active/pending snapshot là requirement.
- Chọn lock khi code ưu tiên dễ chứng minh, transition nhiều hoặc cần condition.
- Chọn single counter/semaphore khi capacity permit mới là invariant cốt lõi.
- Thêm reservation identity khi callback có thể retry/out-of-order.
- Chuyển sang external coordination nếu quota dùng chung giữa nhiều node.

## Lưu ý khi áp dụng thực tế

- Metric: `active`, `pending`, `used`, `limit`, rejection count, CAS retry count,
  handshake timeout/failure và duplicate close.
- Alert khi `used > limit` hoặc counter âm; đây là invariant violation, không chỉ
  là metric bất thường.
- Log transition failure với lifecycle/reservation ID, không log credential.
- Health endpoint phải đọc một state snapshot, không gọi nhiều getter độc lập.
- Giới hạn pending duration; reservation treo phải được timeout và cleanup.
- Không tự động “sửa” counter bằng cách set về limit; điều tra lifecycle leak và
  reconcile với connection owner.
- Khi shutdown, ngừng nhận reservation mới rồi đóng active connection có kiểm
  soát; local state không cần durable rollback.
