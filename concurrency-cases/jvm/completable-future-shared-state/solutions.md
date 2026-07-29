# Solutions và trade-offs

## Value composition theo input order
```java
List<CompletableFuture<Enrichment>> futures = inputs.stream()
    .map(input -> CompletableFuture.supplyAsync(
        () -> client.load(input), executor))
    .toList();
CompletableFuture<Void> all = CompletableFuture.allOf(
    futures.toArray(CompletableFuture[]::new));
return all.thenApply(ignored -> futures.stream()
    .map(CompletableFuture::join)
    .toList());
```
Mỗi worker trả value; chỉ terminal coordinator tạo list. Future list theo input
order nên output không phụ thuộc completion order.

## Failure policy
All-or-nothing: khi deadline/failure, `futures.forEach(f -> f.cancel(true))`, không
trả partial list. Per-item policy: map mỗi child thành sealed `Outcome.Success`/
`Outcome.Failure`, giữ cardinality bằng input và không dùng null để che lỗi.

## Executor và backpressure
Dùng bounded/admission-controlled executor phù hợp blocking dependency; xử lý
`RejectedExecutionException`. Không nested-block trên cùng exhausted pool. Client
có timeout; aggregate có deadline chung.

## Trade-offs
Value composition dễ chứng minh nhưng giữ future/value tới cuối batch. Per-item
outcome rõ partial semantics nhưng API phức tạp hơn. Concurrent queue phù hợp
streaming completion-order, không phù hợp ordered atomic response.

## Production
Immutable output; cancel best-effort; restore interrupt ở blocking boundary;
metric child success/failure/cancel, aggregate latency, queue wait và late
completion sau timeout.
