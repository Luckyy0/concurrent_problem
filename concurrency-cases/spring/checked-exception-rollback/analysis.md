# Phân tích việc phân loại rollback (Rollback classification analysis)

## Trạng thái ban đầu

```text
payout_request:
  id = P-42
  status = RECEIVED
  amount = 300
  wallet_id = W-7
  beneficiary_id = B-BLOCKED

wallet_account:
  id = W-7
  available_balance = 1000

ledger_entry:
  không có PAYOUT_HOLD nào cho P-42
```

Tiến trình điều phối (Dispatcher) chỉ tiến hành lấy các giao dịch chi trả có `status = PROCESSING`. Do danh sách người thụ hưởng bị cấm (Beneficiary registry) đã chứa ID `B-BLOCKED`, chính sách kiểm tra chắc chắn sẽ ném ra ngoại lệ `BeneficiaryRejectedException`.

## Kỳ vọng so với Thực tế

Kỳ vọng khi phương thức ném ra sự từ chối nghiệp vụ (business rejection) đã khai báo:

```text
payout.status       = RECEIVED
available_balance   = 1000
PAYOUT_HOLD count   = 0
dispatchable        = false
transaction outcome = ROLLED_BACK
```

Lỗi thực tế (Broken actual):

```text
payout.status       = PROCESSING
available_balance   = 700
PAYOUT_HOLD count   = 1
dispatchable        = true
transaction outcome = COMMITTED
caller outcome      = BeneficiaryRejectedException
```

Hai kết quả (outcomes) cuối cùng lại mâu thuẫn hoàn toàn với nhau. Caller chỉ nhìn thấy luồng điều khiển thất bại (control-flow failure); trong khi các máy chủ khác lại nhìn thấy trạng thái thành công bền vững trong cơ sở dữ liệu (durable success state).

## Trình tự thời gian của hai tác nhân (Timeline hai actor)

| Bước | Yêu cầu A (Request A) — App node 1 | Tiến trình điều phối B (Dispatcher B) — App node 2 |
| ---: | --- | --- |
| T0 | Proxy mở giao dịch Tx-A | Truy vấn (poll) nhưng chưa thấy P-42 |
| T1 | Khóa các dòng payout và wallet | |
| T2 | Đặt `PROCESSING`, giảm balance `1000 -> 700`, lưu hold | |
| T3 | Chính sách ném ngoại lệ có kiểm tra | |
| T4 | Spring rollback rule đánh giá và trả về `false`; Hibernate có thể đẩy SQL (flush) nếu cần | |
| T5 | Tx-A commit thành công, giải phóng khóa dòng (row locks) | |
| T6 | | Truy vấn mới nhìn thấy P-42 đang `PROCESSING` |
| T7 | Proxy ném lại ngoại lệ ra ngoài; caller nhận kết quả rejected | Tiến hành lấy và điều phối P-42 |

Sự kiện ở mốc T6 hoàn toàn có thể xảy ra trước khi caller kịp xử lý mốc T7. Nguyên nhân là do thao tác commit nằm ẩn bên trong trình chặn (interceptor) và hoàn tất trước khi ngoại lệ được đẩy lại ra bên ngoài proxy.

> **Nói ngắn gọn:** Dữ liệu mới sẽ hiển thị cho mọi người ngay tại thời điểm commit, trong khi caller chỉ nhận được ngoại lệ sau khi thao tác commit này đã xong; tiến trình điều phối có thể hành động trên một trạng thái hoàn toàn trái ngược với kết quả mà API trả về.

## Cội nguồn vấn đề nằm ở tầng Spring

Chú thích `@Transactional` sử dụng một quy tắc rollback (rollback rule) để quyết định kết quả cuối cùng khi phương thức ném ra một ngoại lệ. Theo quy ước mặc định của Spring:

- Giao dịch sẽ rollback nếu gặp `RuntimeException` và các lớp con của nó;
- Giao dịch sẽ rollback nếu gặp `Error` và các lớp con của nó;
- **Giao dịch sẽ không rollback với ngoại lệ có kiểm tra `Exception` theo mặc định.**

Do lớp `BeneficiaryRejectedException extends Exception`, quy tắc mặc định xem đây là một ngoại lệ không đòi hỏi phải rollback. Trình chặn giao dịch (Transaction interceptor) cứ thản nhiên đi tiếp vào luồng commit rồi mới ném lại ngoại lệ gốc ra ngoài.

Quy tắc của trình biên dịch Java (compiler rule) về từ khóa `throws` và quy tắc giao dịch của Spring là hai hệ thống hoàn toàn độc lập. Từ khóa `throws` chỉ bắt buộc người gọi (caller) phải xử lý hoặc khai báo ngoại lệ có kiểm tra đó; nó không hề mang theo dữ liệu siêu mô tả (metadata) rằng "hãy rollback giao dịch này đi".

## Giao dịch vật lý và Phạm vi logic (Physical transaction và logical scope)

Khi hàm `prepare()` không nằm trong một giao dịch bên ngoài (outer transaction), proxy sẽ tạo ra một giao dịch vật lý thực sự và tự động commit nó sau khi gặp ngoại lệ có kiểm tra.

Khi hàm `prepare()` tham gia (join) vào một giao dịch bên ngoài bằng chính sách `REQUIRED`, kết quả cuối cùng sẽ phụ thuộc vào toàn bộ chuỗi gọi hàm (call chain):

- Nếu hàm bên trong (inner) không có `rollbackFor`: ngoại lệ có kiểm tra sẽ không đánh dấu cờ rollback-only;
- Nếu hàm bên ngoài (outer) để cùng loại ngoại lệ có kiểm tra này thoát ra theo quy tắc mặc định: hàm bên ngoài cũng sẽ commit;
- Nếu hàm bên ngoài bắt và nuốt luôn (catch/swallow) ngoại lệ: hàm bên ngoài tiếp tục chạy và có thể commit;
- Nếu hàm bên trong có `rollbackFor`: giao dịch tham gia sẽ được đánh dấu cờ rollback-only;
- Nếu hàm bên ngoài bắt ngoại lệ rồi vẫn cố gắng gọi commit: nó thường sẽ nhận về lỗi `UnexpectedRollbackException`.

Vì vậy, cam kết về việc xử lý khi thất bại (rollback contract) phải được định nghĩa rõ ràng ở ranh giới ngoài cùng, và ngoại lệ bắt buộc phải đi xuyên qua đúng proxy. Bài học từ case SPR-001 vẫn áp dụng cho trường hợp gọi hàm nội bộ (self-invocation).

## Lệnh flush của Hibernate không phải là nguyên nhân

Các đột biến (mutations) trên thực thể bị quản lý (managed entity) có thể chưa sinh ra câu lệnh SQL ngay lập tức. Hibernate thường sẽ kiểm tra trạng thái bẩn (dirty-check) và tự động flush trước khi commit; hoặc một truy vấn khác, hay một lệnh `flush()` thủ công cũng có thể đẩy SQL đi sớm hơn.

Hai biến thể về thời gian:

```text
Flush trễ (late flush): ngoại lệ có kiểm tra -> luồng commit -> flush sinh SQL -> COMMIT
Flush sớm (early flush): SQL đã đẩy đi -> ngoại lệ có kiểm tra -> luồng commit -> COMMIT
```

Nếu quy tắc rollback được cấu hình đúng:

```text
Flush sớm: SQL đã đẩy đi -> ngoại lệ có kiểm tra -> ROLLBACK
```

Những lệnh SQL đã chạy trong giao dịch vẫn sẽ được PostgreSQL đảo ngược hoàn toàn (undo) khi có lệnh rollback. Vì thế, việc gọi `saveAndFlush()` không hề làm cho trạng thái dữ liệu bị "quá muộn để rollback", và loại bỏ nó đi cũng không sửa được bản chất quy tắc phân loại mặc định.

## MVCC Snapshot và Khóa dòng của PostgreSQL (PostgreSQL snapshot và locks)

Mức độ cô lập mặc định `READ COMMITTED` là đủ để chứng minh vấn đề trong case này:

- Giao dịch Tx-A sẽ giữ các khóa dòng (row locks) từ lúc bắt đầu đọc/ghi cho đến tận khi commit hoặc rollback;
- Giao dịch của tiến trình điều phối nếu xảy ra trước mốc T5 sẽ không thể nhìn thấy các thay đổi chưa được commit;
- Nếu truy vấn bắt đầu sau mốc T5, nó chắc chắn nhìn thấy trạng thái `PROCESSING` đã được commit;
- Nếu Tx-A được phép rollback đúng đắn, tiến trình điều phối sẽ không bao giờ nhìn thấy các đột biến trạng thái trung gian (intermediate mutations);
- Cơ sở dữ liệu PostgreSQL hoàn toàn không có khả năng hiểu ngoại lệ của Java mang ý nghĩa nghiệp vụ (business) gì.

Khóa dòng chỉ giải quyết rủi ro ghi đè dữ liệu (write interleaving) trên cùng một dòng. Nó không có thẩm quyền quyết định giao dịch đó sẽ commit hay rollback, và càng không thể ngăn chặn một trạng thái hoàn toàn đúng về mặt cú pháp SQL (nhưng sai bét theo cam kết thất bại nghiệp vụ) trở nên hiển thị cho hệ thống.

## Vì sao đây vẫn được coi là lỗi đồng thời (Concurrency problem)?

Ngay cả một yêu cầu đơn lẻ (single request) cũng đã tạo ra một kết quả mâu thuẫn (inconsistent outcome). Yếu tố đồng thời (concurrency) chỉ làm cho hậu quả vận hành (operational) trở nên rõ ràng và trầm trọng hơn: tiến trình điều phối, yêu cầu thử lại, hệ thống đối soát và các máy chủ khác sẽ phản ứng ngay lập tức với dữ liệu vừa commit từ trước khi con người kịp hiểu ra dòng cảnh báo ngoại lệ trong log.

Trạng thái chia sẻ (Shared state) ở đây không nằm trong bộ nhớ máy ảo Java (JVM) mà nằm ở cơ sở dữ liệu PostgreSQL. Nếu bạn cố bọc một khối `synchronized` quanh hàm `prepare()` trên máy chủ số 1:

- Nó không hề làm thay đổi quy tắc rollback mặc định;
- Nó không thể chặn tiến trình điều phối đang chạy trên máy chủ số 2;
- Nó không thể đảo ngược (undo) được bản ghi commit vĩnh viễn (durable commit);
- Nó không bảo vệ được nếu yêu cầu thử lại vô tình chuyển hướng sang máy chủ khác.

Kết quả của giao dịch cơ sở dữ liệu (Database transaction outcome) và quy tắc kiểm tra trạng thái có thể thực thi (executable-state predicate) mới thực sự là ranh giới phối hợp có uy quyền tuyệt đối (authoritative coordination boundary).

## Việc chủ động bắt ngoại lệ (Catching) làm thay đổi những gì proxy quan sát được

Ví dụ về việc xử lý sai lầm (Broken variant):

```java
@Transactional(rollbackFor = BeneficiaryRejectedException.class)
public PreparationResult prepare(UUID payoutId) {
    try {
        beneficiaryPolicy.verify(...);
        return PreparationResult.ready();
    } catch (BeneficiaryRejectedException rejected) {
        log.info("Rejected payout {}", payoutId);
        return PreparationResult.rejected();
    }
}
```

Do ngoại lệ không hề thoát ra khỏi phương thức, trình chặn (interceptor) sẽ không bao giờ thấy nó để mà áp dụng quy tắc `rollbackFor`. Nếu trong mã lệnh của bạn đã lỡ sửa trạng thái thành có thể thực thi trước khi gọi khối catch và không chịu khôi phục (sửa lại) nó, giao dịch sẽ vẫn vui vẻ tiến hành commit.

Những lựa chọn đáng tin cậy (Credible choices):

- Ném lại (rethrow) để proxy tiến hành rollback;
- Thực hiện kiểm tra tính hợp lệ (validate) từ trước khi thay đổi dữ liệu, rồi sau đó commit kết quả tường minh là `REJECTED`;
- Chủ động đánh dấu cờ rollback-only bằng mã lập trình (programmatically) khi bắt buộc API phải trả về dữ liệu (return);
- Tách riêng bản ghi kiểm toán bị từ chối (rejected audit record) sang một giao dịch độc lập hoàn toàn có chủ đích.

Không thể vừa bắt và nuốt ngoại lệ (swallow exception) lại vừa kỳ vọng cái annotation trên đầu tự động thần giao cách cảm mà rollback giúp bạn.

## Dịch thuật và bao bọc ngoại lệ (Exception translation và wrapping)

Nếu mã nguồn bọc một ngoại lệ có kiểm tra thành một ngoại lệ `RuntimeException`, thì quy tắc rollback mặc định sẽ xảy ra. Tuy nhiên, cấu trúc thứ bậc của các ngoại lệ (exception hierarchy) phải thể hiện rõ cam kết nghiệp vụ (failure contract), chứ không chỉ sinh ra để phục vụ cho sự tiện lợi mặc định của một framework.

Ngược lại, nếu bạn đem bọc một lỗi truy cập dữ liệu không kiểm tra (unchecked data-access failure) vào trong một ngoại lệ có kiểm tra, bạn có thể vô tình biến thao tác rollback thành commit nếu như giao dịch đó chưa được đánh dấu rollback-only. Lỗi `DataAccessException` do Spring chuyển đổi vốn là một lỗi không kiểm tra; đừng làm mất đi tín hiệu cảnh báo quan trọng này khi bạn chuyển tiếp giữa các tầng trừu tượng (abstraction layer).

Nên ưu tiên cấu hình an toàn về kiểu (type-safe):

```java
rollbackFor = BeneficiaryRejectedException.class
```

Việc dùng chuỗi tên lớp (String-based class-name rules) rất dễ bị khớp nhầm (match rộng) hoặc báo lỗi nếu tên ngoại lệ trùng nhau một phần. Nếu bạn sử dụng các siêu chú thích dùng chung (shared meta-annotation), thì bài kiểm thử tích hợp (integration test) bắt buộc phải chứng minh được hành vi thực tế (effective behavior), chứ không chỉ dùng để soi xem có annotation hay không.

## Thao tác commit cũng có thể thất bại

Việc quy tắc mặc định quyết định "hãy commit đi" không đồng nghĩa với việc commit chắc chắn thành công. Quá trình flush có thể vi phạm điều kiện dữ liệu duy nhất (unique constraint), hết thời gian chờ khóa (lock timeout) hoặc rớt kết nối; khi đó, người gọi có thể sẽ nhận lại một lỗi hoàn toàn khác (transaction/data-access exception).

Case này giả định việc commit thành công là để cô lập phân tích riêng vấn đề phân loại ngoại lệ rollback. Việc hết thời gian gọi I/O từ xa hay không rõ kết quả của hệ thống ngoài đều nằm ngoài phạm vi phân tích.

## Thử lại và Lệnh trùng lặp (Retry và duplicate command)

Caller khi thấy ngoại lệ có kiểm tra có thể sẽ thử gọi lại (retry) P-42 hoặc phát một lệnh hoàn toàn mới:

- Nếu gọi lại cùng ID, nó sẽ bị vấp phải trạng thái `PROCESSING`, thay vì trạng thái ban đầu `RECEIVED`;
- Nếu gọi ID mới, hệ thống có thể sẽ giữ tiền thêm một lần nữa nếu không có ràng buộc về tính lũy đẳng (idempotency constraint);
- Một khóa duy nhất của sổ cái (unique ledger key) có thể chặn được bản ghi trùng lặp nhưng lại không thể tự động đảo ngược trạng thái đang `PROCESSING` của đơn hàng;
- Việc nhắm mắt nhắm mũi thử lại (retry mù) sẽ không sửa được kết quả bền vững đã lỡ sai ở lần thử đầu tiên.

Tính lũy đẳng (Idempotency) vẫn rất cần thiết để xử lý quá trình chuyển giao lệnh (command delivery), nhưng nó không sinh ra để thay thế quy tắc rollback (rollback rule). Hai cơ chế này bảo vệ cho hai rào chắn tính đúng đắn hoàn toàn khác nhau.

## Hành vi khi sự cố xảy ra (Crash behavior)

- Sập trước khi PostgreSQL kịp commit: Việc mất kết nối (connection close) sẽ khiến giao dịch bị rollback.
- Sập sau khi đã commit nhưng trước khi trả về (API response): Dữ liệu đã bền vững nhưng người gọi không rõ kết quả; đây lại là bài toán mơ hồ về kết quả phản hồi.
- Trong trường hợp lỗi do checked exception: Tiến trình không cần phải sập; proxy sẽ tự động commit êm đẹp rồi ném ra ngoại lệ.
- Tiến trình điều phối (Dispatcher) sập sau khi đã giữ công việc thì cần cơ chế khôi phục và lũy đẳng độc lập; việc đó không hề làm thay đổi nguyên nhân cốt lõi của SPR-005.

## Khả năng quan sát (Observability)

Nên luôn ghi nhận kèm theo các bộ định danh tương quan (correlation identifiers):

- ID chi trả (payout ID), mã khóa lệnh lũy đẳng (command/idempotency key) và tên giao dịch;
- Kết quả xử lý nghiệp vụ `READY` / `REJECTED`;
- Trạng thái hoàn tất của giao dịch thu thập từ các công cụ chuẩn đoán/kiểm thử (test/diagnostic instrumentation);
- Đối chiếu trạng thái đã commit của giao dịch chi trả với bản ghi trong sổ cái;
- Việc lấy việc của tiến trình điều phối diễn ra sau sự từ chối;
- Lỗi `UnexpectedRollbackException` nếu phạm vi bên ngoài cố tình nuốt chửng việc đòi rollback của phạm vi bên trong.

Không ghi log (log) toàn bộ dữ liệu tài khoản hay ngân hàng. Cảnh báo quan trọng nhất chính là sự vênh nhau (mismatch) giữa kết quả nghiệp vụ bị từ chối và trạng thái tài chính vẫn ngang nhiên có thể thực thi dưới cơ sở dữ liệu, chứ không chỉ đơn thuần là đếm số lượng ngoại lệ.
