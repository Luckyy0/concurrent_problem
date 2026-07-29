# Broken implementation

```java
List<Enrichment> results = new ArrayList<>();
List<CompletableFuture<Void>> tasks = inputs.stream()
    .map(input -> CompletableFuture
        .supplyAsync(() -> client.load(input), executor)
        .thenAccept(results::add))
    .toList();
CompletableFuture.allOf(tasks.toArray(CompletableFuture[]::new)).join();
return results;
```

Callbacks có thể chạy đồng thời; `ArrayList.add` không thread-safe. `allOf` không
serialize callbacks và không biến accumulator thành atomic. Nếu một child fail,
`join` ném `CompletionException`, nhưng child khác có thể vẫn chạy và mutate list.
Trả/cached reference mutable còn cho phép state thay đổi sau response.

## Điều kiện tái hiện
Ít nhất hai callback overlap; shared accumulator bị capture; một failure/timeout
không cancel mọi sibling; caller quan sát list trước lifecycle kết thúc.

## Các cách sửa chưa đủ
`Collections.synchronizedList` chỉ sửa add, không định nghĩa order/failure;
`CopyOnWriteArrayList` đắt cho write-heavy; `allOf` không fail-fast/cancel sibling;
`exceptionally` trả null có thể che failure; dùng common pool cho blocking I/O có
thể nối sang starvation của JVM-008.

> **Nói ngắn gọn:** thread-safe collection không quyết định batch là all-or-nothing
hay per-item outcome.
