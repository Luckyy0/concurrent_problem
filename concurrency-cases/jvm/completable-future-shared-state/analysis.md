# Interleaving và root cause

## Timeline
T1/T2 cùng hoàn tất, cùng đọc internal size của list và ghi; một result có thể bị
mất. T3 fail làm aggregate exceptional; T2 vẫn hoàn tất sau đó và mutate partial
state.

## Expected và actual
Expected immutable, đủ, ordered success hoặc explicit failure. Actual list thiếu,
completion-order, thay đổi sau exception và cancellation không propagate.

## Root cause
Completion stages độc lập về scheduling. Happens-before từ completion tới `join`
chỉ bảo đảm visibility của state cuối đã ghi; nó không sửa concurrent mutation.
`allOf` là completion barrier, không phải transaction hoặc structured cancellation.

## Failure, timeout và cancellation
Dùng deadline chung; khi aggregate fail/timeout, cancel children best-effort.
Remote client vẫn cần timeout. Không publish accumulator trước terminal outcome.
`join` bọc lỗi trong `CompletionException`; unwrap cause có chủ đích.

## Multi-instance
State chỉ trong JVM; async stage không mang Spring transaction context mặc định.
Transaction context loss thuộc SPR-002.

> **Nói ngắn gọn:** composition graph phải sở hữu cả value flow lẫn failure flow;
shared side effect đứng ngoài graph sẽ thoát khỏi completion contract.
