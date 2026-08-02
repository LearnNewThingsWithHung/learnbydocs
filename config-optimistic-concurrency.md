- Khi chúng ta thực hiện câu lệnh Update hay Insert trong PostgreSQL, PostgreSQL không ghi đè dữ liệu trực tiếp. Thay vào đó, nó sử dụng một cơ chế gọi là **MVCC**(Multi-Version Concurrency Control)

	Mỗi khi bạn thêm mới (**Insert**) hoặc sửa đổi (**Update**) một dòng, PostgreSQL tạo ra một "**row version**"(phiên bản dòng) mới.

	Mọi dòng trong mọi bảng của PostgreSQL đều có sẵn các cột hệ thống ẩn(system columns) mà bình thường câu lệnh **SELECT** không hiểu thị, tiêu biểu trong số đó là **xmin**

	**xmin** lưu trữ **TransactionID**(XID) của transaction tạo ra hoặc cập nhật dòng dữ liệu đó.
	**XID** là một số nguyên tăng dần 32-bit  (unit) quản lý bởi Postgres Engine.

	**Đặc điểm quan trọng**: Mỗi khi có câu lệnh Update tác động vào  dòng đó thành công, Postgres tự động cập nhật lại giá tri của cột **xmin** này thành ==TransactionID mới nhất.== 

Tóm tắt lại ngắn gọn phía trên:
Mỗi sự thay đổi đều sinh ra một **row version mới**, kiểm tra concurrency bằng cách dựa trên row version đó với sự thay đổi của **transactionId** là "**xid**" của column là "**xmin**"(một column ẩn của Postgres)

Tình hướng thực tế:

1. Lúc 08:00: UserA và UserB cùng đọc thông tin thấy **Ticket có Id = 1**
	- Postgres trả về : `Title = Lỗi login`, `xmin = 1005`(transactionID là 1005)
	- Cả EF của UserA và UserB đề đang row version với xmin = 1005 trong memory(ChangeTracker)
2. Lúc 08:01: UserA đổi `Title` thành `Lỗi login trên Mobile` và ấn lưu
- FE thực hiện câu lệnh
```SQL
UPDATE tickets
SET title = 'Lỗi login trene Mobile'
Where id = 1 AND xmin = 1005; -> so sánh xmin
```
- Kết quả: Tìm tấy 1 dòng khớp cả id = 1 và xmin = 1005 -> Lưu thành công
- Hệ quả: Postgres đổi xmin của dòng id = 1 thành xmin = 1006

3. Lúc 08:02: UserB đổi Title thành "Lỗi đăng nhập" và bấm lưu
- EF core của UserB vẫn đang giữ `xmin = 1005` từ lúc 08:00, nên nó sẽ tiếp lực gửi câu lệnh
```SQL
UPDATE tickets 
SET title = 'Lỗi đăng nhập'
WHERE id = 1 AND xmin = 1005; -- <--- Vẫn tìm xmin cũ = 1005
```
- Kết quả: Vì `xmin` ở Postgres hiện tại đã là 1006, câu lệnh Where ... trả về 0 dòng bị ảnh hưởng (0 rows affected).
- Phản ứng của EF Core: EF Core kỳ vọng sửa được 1 dòng, nhuwg kết quả trả về là 0 dòng. EF Core lập tức hủy Transcation và quăng ra ngoại lệ: `DbUpdateConcurrencyException`