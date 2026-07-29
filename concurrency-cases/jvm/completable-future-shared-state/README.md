# JVM-009 — CompletableFuture và shared aggregation race

## Tóm tắt
Nhiều stage hoàn tất trên các thread khác nhau rồi cùng `add` vào một `ArrayList`.
Một stage fail trong khi stage khác vẫn mutate, khiến caller thấy partial result.

Invariant: mỗi input có đúng một outcome; final success chỉ được publish sau khi
mọi stage success; failure/cancel không trả mutable partial list; output giữ input
order khi API yêu cầu.

> **Nói ngắn gọn:** future là container của completion, không phải lock cho object
được nhiều callback capture.

## Thuật ngữ cần biết
| Thuật ngữ | Giải thích |
| --- | --- |
| completion stage | Bước chạy khi future hoàn tất |
| value composition | Ghép value trả về thay vì mutate state ngoài pipeline |
| allOf | Barrier hoàn tất khi các future con kết thúc |
| exceptional completion | Future kết thúc bằng exception |
| confinement | Chỉ coordinator mutate accumulator |
| partial visibility | State dở dang bị caller quan sát |

## Bối cảnh và contention point
Batch enrichment fan-out profile, price và risk calls. Callback dùng shared list;
`allOf`, callback error và cancellation cạnh tranh trên accumulator.

## Điều hướng
- [Broken code](broken-code.md)
- [Phân tích](analysis.md)
- [Solutions](solutions.md)
- [Experiments](experiments.md)
- [JMM](../../concepts/java-memory-model-and-atomicity.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả và hướng sửa
Mất/trùng result, sai order, response partial và task nền tiếp tục sau timeout.
Để mỗi future trả value; sau `allOf`, một coordinator đọc future theo input order
và tạo immutable list. Deadline/cancel/failure policy phải explicit.
