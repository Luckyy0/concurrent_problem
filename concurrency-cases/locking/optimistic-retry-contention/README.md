# LOCK-002 — Bounded optimistic retry dưới contention

## Tóm tắt

Nhiều command cộng reward points vào cùng wallet `@Version`. Mỗi command là delta
trên fresh state nên có thể retry, nhưng immediate/unbounded retry làm losers cùng
reload–flush, tiếp tục conflict và khuếch đại database load.

Retry đúng:

```text
attempt Tx-N load current wallet + command record
→ apply delta + flush
→ conflict: rollback hoàn tất
→ check attempt/deadline
→ backoff có jitter ngoài transaction
→ attempt Tx-(N+1) reload
```

> **Nói ngắn gọn:** `@Version` giữ correctness; bounded fresh retry giữ hệ thống
> không biến conflict thành retry storm.

## Actor và trạng thái dùng chung

| Thành phần | Trạng thái |
| --- | --- |
| `reward_wallet` | wallet `77`, points `100`, version `10` |
| C1…Cn | Unique command ID và positive point delta |
| `reward_credit` | Durable idempotency/audit record theo command ID |
| App-1…App-N | Concurrent Spring instances |

Điểm tranh chấp là versioned UPDATE của cùng wallet row. Retry không giữ row lock
trong backoff nhưng mỗi attempt vẫn dùng connection/query/flush.

## Invariant

```text
Final points = initial points + tổng delta của các unique committed commands.

Mỗi command ID tạo tối đa một reward_credit.

Mỗi retry dùng transaction/persistence context mới và reload wallet.

Retry dừng theo attempt cap hoặc overall deadline; exhaustion không báo success.
```

## Ranh giới transaction

`RewardCreditCoordinator` không có transaction. Nó gọi
`RewardCreditAttempt.creditOnce()` qua bean proxy. Attempt dùng
`REQUIRES_NEW`, kiểm tra command replay, load wallet, apply delta, insert credit
record, flush và commit/rollback.

Backoff chạy sau proxy rollback, nên không giữ connection. Caller chỉ nhận result
sau successful commit.

## Khi retry an toàn?

Command `add 10 points` được tính lại trên current balance và cùng `commandId`, nên
retry có thể an toàn. Không suy rộng sang absolute user edit như “set price=80”,
remote side effect không idempotent hoặc business rejection.

Mỗi attempt phải revalidate:

- command đã commit chưa;
- wallet còn active;
- delta/policy còn hợp lệ;
- deadline/cancellation còn cho phép.

## Thuật ngữ cần biết

| Thuật ngữ | Ý nghĩa trong case |
| --- | --- |
| retry amplification | Một request tạo nhiều attempts/queries/writes |
| retry storm | Nhiều losers retry đồng nhịp và tiếp tục conflict |
| fresh attempt | Transaction, snapshot và persistence context mới |
| bounded retry | Attempt cap cộng overall deadline |
| exponential backoff | Delay tăng theo attempt |
| jitter | Thành phần random làm actors bớt đồng nhịp |
| starvation | Command liên tục thua tới exhaustion |
| idempotency record | Durable row bảo đảm cùng command không cộng điểm lần hai |
| exhaustion | Retry budget hết mà command chưa commit |

## Điều hướng

- [Code retry sai và load amplification](broken-code.md)
- [Timeline, rollback, fairness và crash](analysis.md)
- [Coordinator/attempt, policy và backoff](solutions.md)
- [PostgreSQL Testcontainers experiments](experiments.md)
- [Optimistic locking và version conflict](../../concepts/optimistic-locking.md)
- [Ranh giới transaction trong Spring](../../concepts/spring-transaction-boundaries.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong production

- CPU/query/flush tăng nhanh hơn request rate;
- connection pool bị chiếm bởi attempts liên tiếp;
- tail latency và exhaustion tăng trên hot wallet;
- same-transaction retry dùng rollback-only context và không tiến triển;
- tạo command ID mới mỗi attempt làm duplicate credits;
- retry đồng nhịp gây starvation cho một số command;
- response mất sau commit tạo ambiguous outcome nếu không replay by command ID.

## Hướng sửa khuyến nghị

1. Chỉ retry allowlisted optimistic conflict.
2. Tách non-transactional coordinator và one-attempt worker bean.
3. Mỗi attempt reload aggregate/command state.
4. Giữ nguyên command ID và unique constraint.
5. Dùng attempt cap, overall deadline, exponential backoff có jitter.
6. Ghi attempts, success-after-retry, exhaustion và conflict rate.
7. Khi contention cao bền vững, đổi strategy thay vì chỉ tăng attempts.

## Phạm vi

Case dành cho low/moderate contention và retry-safe command. Detection cơ bản ở
`LOCK-001`; strategy dưới high contention thuộc `LOCK-005`; Spring advisor
ordering chi tiết ở `SPR-006`.
