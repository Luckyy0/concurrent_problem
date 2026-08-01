# DB-005 — Write skew trên quy tắc nhiều row

## Tóm tắt

Danh sách trực ca đêm `NOC-NIGHT-42` có hai nhân viên đang on-call: Alice và Bob. Mỗi người gửi yêu cầu rời ca trong một Spring transaction riêng. Cả hai cùng đọc `onCallCount = 2`, kết luận người còn lại vẫn trực, rồi cập nhật row của chính mình thành `on_call = false`.

Hai thao tác UPDATE ghi hai rows khác nhau nên không có xung đột ghi trực tiếp. Cả hai có thể commit; danh sách trực cuối cùng không còn nhân viên nào.

Quy tắc bất biến (Invariant):

```text
Với mỗi danh sách trực đang hoạt động:
  count(assignments where on_call = true) >= 1

Ban đầu: Alice=true, Bob=true
Mong đợi: chỉ một yêu cầu được chấp nhận
Kết quả sai: Alice=false, Bob=false
```

> **Nói ngắn gọn:** mỗi thao tác cập nhật row riêng lẻ đều hợp lệ, nhưng tổ hợp hai commits làm sai quy tắc của cả tập rows; đó là write skew.

## Thành phần và trạng thái chia sẻ

| Thành phần | Vai trò |
| --- | --- |
| Transaction A của Alice | Đọc danh sách trực rồi tắt on-call của Alice |
| Transaction B của Bob | Đọc cùng danh sách trực rồi tắt on-call của Bob |
| `on_call_roster` | Thực thể cha/danh sách trực ổn định |
| `on_call_assignment` | Một row cho mỗi nhân viên trong danh sách trực |
| Hibernate | Kiểm tra thay đổi (dirty-check) entity và thực thi versioned UPDATE |
| PostgreSQL MVCC | Cấp snapshot và lock từng row được cập nhật |

Các rows đã commit ban đầu:

```text
roster=NOC-NIGHT-42
Alice: on_call=true, version=0
Bob:   on_call=true, version=0
```

Các rows bị lỗi sau cùng:

```text
Alice: on_call=false, version=1
Bob:   on_call=false, version=1
```

## Ranh giới transaction và điểm tranh chấp

Mỗi lời gọi `leaveOnCall()` đi qua Spring proxy:

```text
BEGIN
  SELECT COUNT(*) WHERE roster_id=? AND on_call=true
  if count <= 1 -> reject
  SELECT own assignment
  set own on_call=false
  UPDATE own row WHERE version=?
COMMIT
```

Chuỗi thực thi không nguyên tử:

```text
read multi-row invariant -> decide -> update one member row
```

Điểm tranh chấp logic là toàn bộ phân công của danh sách trực. Quá trình ghi vật lý lại là row của Alice và row của Bob; các row locks cùng các điều kiện phiên bản lạc quan (optimistic version predicates) không chạm trán nhau.

## Kết quả mong đợi và thực tế

| | A/Alice | B/Bob | Kết quả cuối |
| --- | --- | --- | --- |
| Snapshot count | 2 | 2 | |
| Quyết định | cho phép rời ca | cho phép rời ca | |
| Row được cập nhật | Alice | Bob | |
| Số rows bị ảnh hưởng bởi `@Version` | 1 | 1 | |
| Kết quả lỗi | commit | commit | 0 on-call |
| Kết quả yêu cầu | thành công | từ chối/thử lại | 1 on-call |

## Sắc thái cô lập (Isolation)

- `READ COMMITTED`: hai câu lệnh COUNT có thể cùng chạy trước khi ghi và thấy `2`; cả hai commits đều có thể xảy ra.
- PostgreSQL `REPEATABLE READ`: mỗi phía tương tác giữ một snapshot ổn định thấy `2`; các thao tác cập nhật row khác nhau vẫn có thể cùng commit. Snapshot ổn định không bảo vệ quy tắc liên kết nhiều rows (cross-row invariant).
- `SERIALIZABLE`: SSI theo dõi các phụ thuộc điều kiện và có thể hủy (abort) một tiến trình với SQLSTATE `40001`; ứng dụng cần thử lại có giới hạn (bounded retry).

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| write skew | Các transactions đọc cùng quy tắc rồi ghi các rows khác nhau làm quy tắc bị sai |
| snapshot isolation | Mỗi transaction ra quyết định từ một snapshot đã commit ổn định |
| multi-row invariant | Quy tắc phụ thuộc nhiều rows thay vì một row riêng lẻ |
| rw-antidependency | Phía đọc không thấy dữ liệu ghi sau đó của transaction khác |
| SSI | Serializable Snapshot Isolation của PostgreSQL |
| guard row | Row chia sẻ mà mọi transaction làm thay đổi quy tắc phải lock/cập nhật |
| serialization failure | Lỗi có thể thử lại với SQLSTATE `40001` |
| optimistic version | Điều kiện phiên bản phát hiện cập nhật đồng thời trên cùng row |

## Điều hướng

- [Cài đặt versioned-row bị lỗi](broken-code.md)
- [Phân tích snapshot và dependency](analysis.md)
- [Các giải pháp guard row, locking và serializable](solutions.md)
- [Các thử nghiệm PostgreSQL tất định](experiments.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Các mức độ cô lập (Isolation levels)](../../concepts/isolation-levels.md)
- [Kiểm thử đồng thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## Hậu quả trên môi trường production

- Ca trực không còn người chịu trách nhiệm dù cả hai yêu cầu đều trả về thành công.
- Cảnh báo/leo thang sự cố không có người xử lý; SLA và thời gian phản hồi sự cố bị ảnh hưởng.
- Kiểm toán từng row không cho thấy bản ghi cập nhật lỗi hay ngoại lệ (exception).
- Thử lại yêu cầu trùng lặp không sửa được quy tắc đã commit sai.
- Nhiều thực thể ứng dụng làm cho việc điều phối cục bộ trong JVM vô hiệu.
- Đối soát chỉ phát hiện sau khi danh sách trực đã rơi vào trạng thái không an toàn.

## Hướng sửa khuyến nghị

Với một thực thể danh sách trực cha ổn định, ưu tiên lock guard row bằng `FOR UPDATE`, rồi đếm và cập nhật trong cùng một transaction ngắn. Mọi luồng làm thay đổi quy tắc phải theo cùng một giao thức.

Nếu mô hình tranh chấp/thử lại phù hợp, `SERIALIZABLE` cùng với việc thử lại toàn bộ transaction là lựa chọn tốt cho điều kiện quy tắc phức tạp. Một `on_call_count` chính thức với thao tác giảm có điều kiện cũng có thể chuyển quy tắc thành xung đột trên một row.

`@Version` trên phân công vẫn hữu ích cho lỗi mất cập nhật (lost update) trên cùng row, nhưng không thay thế guard/SSI cho quy tắc nhiều rows.

## Phạm vi

Trường hợp này chỉ xét write skew khi các transactions cập nhật các rows có sẵn khác nhau. Phantom inserts/sức chứa thuộc DB-004. Cơ chế lock tổng quát được đào sâu ở DB-007 và thứ tự deadlock ở DB-008.
