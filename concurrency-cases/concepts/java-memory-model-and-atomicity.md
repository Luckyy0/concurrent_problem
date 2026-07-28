# Java Memory Model, atomicity và shared mutable state

## Mục đích

Tài liệu này là nền tảng dùng chung cho các case có nhiều thread truy cập cùng
một object trong một JVM. Nó phân biệt ba thuộc tính thường bị đánh đồng:

- **atomicity**: operation có bị xen kẽ giữa chừng hay không;
- **visibility**: một thread có nhìn thấy write của thread khác hay không;
- **ordering**: compiler, JVM và CPU được phép quan sát các operation theo thứ
  tự nào.

Một chương trình có visibility đúng vẫn có thể sai atomicity. Ví dụ,
`volatile long counter` làm write dễ quan sát hơn nhưng `counter++` vẫn là
compound operation:

```text
read counter → add 1 → write counter
```

## Java Memory Model và happens-before

Java Memory Model (JMM) định nghĩa các giá trị mà một read được phép quan sát và
quan hệ **happens-before** giữa các action. Một số quan hệ quan trọng:

- unlock một monitor happens-before lần lock tiếp theo trên cùng monitor;
- write vào một `volatile` field happens-before read tiếp theo của field đó;
- action trước khi gọi `Thread.start()` happens-before action trong thread mới;
- mọi action của thread hoàn tất happens-before thread khác trở về từ
  `Thread.join()`;
- action được nối theo program order trong cùng một thread.

`happens-before` tạo visibility và ordering cần thiết. Nó không tự biến một
chuỗi nhiều operation thành một atomic operation.

## Atomic operation và compound invariant

Read/write của một reference riêng lẻ có thể atomic nhưng invariant thường bao
gồm nhiều bước hoặc nhiều field:

```java
if (remaining > 0) {
    remaining--;
}
```

Invariant `remaining >= 0` trải qua `read → decide → write`. Muốn bảo vệ nó,
toàn bộ chuỗi phải dùng cùng một coordination mechanism, chẳng hạn:

- một monitor qua `synchronized`;
- một `Lock` được mọi actor dùng nhất quán;
- một atomic compare-and-set loop;
- confinement: chỉ một thread sở hữu mutable state;
- loại bỏ shared mutable state.

Một thread-safe field không làm cho invariant nhiều field trở thành thread-safe.
Ví dụ `AtomicLong sequence` không bảo vệ cặp `(sequence, customerId)` nếu
`customerId` vẫn là shared mutable field.

## volatile, atomics và locks

| Cơ chế | Bảo đảm chính | Không tự bảo đảm |
| --- | --- | --- |
| `volatile` | visibility và ordering quanh một field | compound atomicity |
| `AtomicLong` | atomic operation/CAS trên một value | invariant nhiều field |
| `synchronized` | mutual exclusion và happens-before trên cùng monitor | coordination giữa nhiều JVM |
| `ReentrantLock` | mutual exclusion cùng các policy như interrupt/timeout | correctness nếu actor dùng lock khác |
| immutable object | state không đổi sau safe publication | atomicity của workflow bên ngoài object |

Ưu tiên thiết kế **stateless** hoặc immutable trước khi thêm lock. Khi không còn
shared mutable state, không còn critical section phải phối hợp.

## Safe publication

Object chỉ thực sự immutable khi:

1. state hoàn chỉnh được thiết lập trước khi publish;
2. field phù hợp là `final`;
3. reference không thoát khỏi constructor;
4. publication đi qua một happens-before edge hợp lệ.

Spring publish singleton bean an toàn sau khi khởi tạo, nhưng điều đó không làm
các mutable field được cập nhật trong lúc xử lý request trở thành thread-safe.

## Ranh giới JVM và distributed system

Mỗi application instance có heap, monitor và atomic variable riêng:

```text
Load Balancer
    ├── App A: counter = 42, lock A
    └── App B: counter = 42, lock B
```

`synchronized`, `ReentrantLock` và `AtomicLong` ở App A không phối hợp với App
B. Invariant dùng chung giữa các node phải được bảo vệ tại authoritative shared
boundary, ví dụ database constraint, conditional SQL, row lock, durable
idempotency record hoặc một protocol lease/fencing phù hợp.

## Liên kết

- Case áp dụng trực tiếp: [JVM-001](../jvm/spring-singleton-mutable-state/README.md)
- Kỹ thuật kiểm thử: [Concurrency testing](concurrency-testing.md)
- Các case liên quan được lập kế hoạch trong
  [catalog](../CONCURRENCY_CASE_LIBRARY.md): `JVM-005`, `JVM-006`, `DB-001`
