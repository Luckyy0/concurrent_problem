# Deterministic experiments

## Broken race
Dùng hai latch để giữ callbacks, rồi cho cả hai gọi hooked accumulator tại cùng
logical index; assert size 1 thay vì 2. Với `ArrayList` thật, chạy stress suite
riêng—không coi một lần pass là bằng chứng.

## Fixed regression
```java
@Test
void composesValuesInInputOrder() {
    CompletableFuture<String> first = new CompletableFuture<>();
    CompletableFuture<String> second = new CompletableFuture<>();
    List<CompletableFuture<String>> futures = List.of(first, second);
    CompletableFuture<List<String>> aggregate = CompletableFuture
        .allOf(first, second)
        .thenApply(x -> futures.stream().map(CompletableFuture::join).toList());
    second.complete("B");
    first.complete("A");
    assertEquals(List.of("A", "B"),
        aggregate.orTimeout(5, TimeUnit.SECONDS).join());
}
```

## Failure/cancellation
Cho một child fail, một child block trên latch; assert aggregate exceptional,
cancel sibling và không có final list được publish. Mọi latch/future có timeout;
không dùng `Thread.sleep`.

## Production verification
Metric input/success/failure/cancel/late-completion, aggregate deadline, executor
queue/rejection. Assert cardinality, unique input ID, order và immutable output.

## Checklist
- [ ] Callback overlap được điều phối bằng latch.
- [ ] Fixed output giữ input order dù completion ngược.
- [ ] Failure không publish partial result.
- [ ] Cancellation và timeout được test.
- [ ] Executor/rejection được quan sát.
