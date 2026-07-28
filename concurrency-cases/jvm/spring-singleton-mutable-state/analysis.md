# Race timeline và root cause

## Initial state

```text
nextSequence  = 41
lastCustomerId = null

T1 input = "alice"
T2 input = "bob"
```

Expression `++nextSequence` được mô hình hóa thành ba bước:

```text
read nextSequence → add 1 → write nextSequence
```

## Interleaving

```text
T1: alice                                  T2: bob
----------------------------------------   ----------------------------------
lastCustomerId = "alice"
read nextSequence -> 41
                                             lastCustomerId = "bob"
                                             read nextSequence -> 41
                                             add 1 -> 42
                                             write nextSequence = 42
add 1 -> 42
write nextSequence = 42
read lastCustomerId -> "bob"
return ReceiptDraft(42, "bob")
                                             read lastCustomerId -> "bob"
                                             return ReceiptDraft(42, "bob")
```

## Expected và actual

```text
Expected:
  alice -> ReceiptDraft(42, "alice")
  bob   -> ReceiptDraft(43, "bob")
  unique sequences = 2

Actual:
  alice -> ReceiptDraft(42, "bob")
  bob   -> ReceiptDraft(42, "bob")
  unique sequences = 1
```

Hai invariant cùng bị phá:

- lost update làm sequence `42` được cấp hai lần;
- customer của T1 bị thay bằng dữ liệu của T2.

## Root cause theo layer

### Spring bean lifecycle

`@Service` mặc định có singleton scope trong một `ApplicationContext`. Spring
không tạo service mới cho mỗi HTTP request và không serialize method call.
Safe publication lúc bean khởi tạo không bảo vệ mutable writes trong lúc chạy.

### JVM và Java Memory Model

Có data race vì hai thread access cùng field, ít nhất một access là write và
không có synchronization/happens-before relationship.

Hai lỗi độc lập:

1. `++nextSequence` là non-atomic read-modify-write;
2. `lastCustomerId` là request data được lưu trong shared field rồi đọc lại sau
   một cửa sổ interleaving.

Root cause chính xác là:

```text
shared field write → interleaving → compound update/read shared field
```

Không phải chỉ vì “có nhiều request cùng lúc”.

### Spring transaction, Hibernate và database

Case này không access database, không có persistence context, MVCC hay database
lock. Ngay cả nếu method có `@Transactional`, mỗi request có transaction riêng
nhưng cùng Java object. Database rollback cũng không rollback giá trị Java
field.

## Commit, rollback, timeout và crash

- **Commit/rollback:** không áp dụng cho local fields; không có transaction log
  phục hồi chúng.
- **Exception:** nếu method lỗi sau khi tăng sequence, sequence bị bỏ trống;
  không tự rollback.
- **Timeout/retry:** client retry tạo một draft mới; đây không phải idempotent
  workflow.
- **Process crash/restart:** local counter trở về initial value và có thể tái sử
  dụng sequence cũ.

Các đặc tính này cho thấy local counter không phù hợp làm durable business ID.

## Multi-instance behavior

Nếu production có hai application instance:

```text
App A: nextSequence = 41, monitor/AtomicLong A
App B: nextSequence = 41, monitor/AtomicLong B
```

Ngay cả implementation dùng `synchronized` hoặc `AtomicLong`, hai node vẫn có
thể cấp cùng sequence. Local coordination chỉ có hiệu lực trong JVM sở hữu nó.

## Consequences

### Technical

- duplicate sequence và lost update;
- cross-request data leakage;
- log/correlation không đáng tin;
- behavior thay đổi theo scheduler;
- không có recovery contract sau crash.

### Business

- customer nhận draft chứa identifier của customer khác;
- downstream dedup/correlation có thể gộp nhầm operation;
- audit và incident reconstruction sai;
- rủi ro riêng tư nếu shared field chứa dữ liệu nhạy cảm.

## Vì sao lỗi khó tái hiện bằng unit test thường

Một unit test tuần tự tạo total order:

```text
call A hoàn tất → call B bắt đầu
```

Nó không mở cửa sổ sau `read` và trước `write`. Regression test cần barrier/latch
để điều phối interleaving; xem [experiments](experiments.md).

