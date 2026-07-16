 #### 1. Audit Logs (Ghi vết dữ liệu tự động)

Để hiểu tại sao Interceptor lại làm được việc này một cách kỳ diệu, trước tiên bạn cần hiểu về **Change Tracker** của EF Core.

##### Cơ chế hoạt động ở tầng dưới (Under the hood):

Khi bạn thêm mới hoặc sửa đổi một đối tượng dữ liệu trong code, EF Core chưa lao vào Database ngay. Nó sẽ giữ đối tượng đó trong bộ nhớ và gắn cho nó một cái "nhãn" (State):

- Nếu gọi `_context.Add(user)` $\rightarrow$ Nhãn là `Added`.
    
- Nếu thay đổi thuộc tính `user.Name = "New Name"` $\rightarrow$ Nhãn là `Modified`.
    

Khi bạn bấm nút chốt hạ `_context.SaveChanges()`, EF Core kích hoạt một chuỗi sự kiện. **SaveChangesInterceptor** sẽ nhảy vào đúng ngã rẽ ngay trước khi câu lệnh SQL được biên dịch.

```
[Ứng dụng gọi SaveChanges] 
          │
          ▼
[Interceptor đánh chặn (SavingChanges)] ──► Quét Change Tracker ──► Tự động điền Audit Fields
          │
          ▼
[EF Core dịch thành SQL] ──► [Gửi xuống Database]
```

##### Tại sao ứng dụng này lại "tối thượng" trong dự án thực tế?

Nếu không dùng Interceptor, mỗi khi viết hàm cập nhật thông tin, bạn sẽ phải làm thế này:

C#

```CSharp
// Code thủ công - ác mộng bảo trì
public void UpdateProduct(Product prod) {
    prod.UpdatedAt = DateTime.UtcNow;
    prod.UpdatedBy = _currentUserService.GetCurrentUserId(); // Lặp đi lặp lại ở 100 chỗ khác nhau
    _context.Products.Update(prod);
    _context.SaveChanges();
}
```

**Khi có Interceptor:** Bạn hoàn toàn "giải phóng" bộ não của lập trình viên.

- Interceptor ép các Entity thực thi một `interface` chung (ví dụ `IAuditable`).
    
- Khi chạy qua bộ chặn, nó dùng kỹ thuật **Reflection** hoặc ép kiểu để tự động kiểm tra: _"À, anh này là `IAuditable` và đang bị `Modified` đúng không? Tôi sẽ tự điền `UpdatedAt = Now` và `UpdatedBy = User hiện tại`"_.
    
- Lập trình viên tầng nghiệp vụ (Business Logic) chỉ tập trung viết logic sửa tên, sửa giá, tuyệt đối không cần quan tâm đến các trường Audit này nữa.
#### 2. Xử lý Soft Delete (Xóa mềm) tự động

Xóa mềm (Soft Delete) là một kỹ thuật bắt buộc trong các hệ thống lớn để tránh mất mát dữ liệu do sai sót của con người hoặc để phục vụ mục đích thống kê, phục hồi sau này. Tuy nhiên, nếu làm thủ công, nó cực kỳ dễ lỗi.

##### Thử thách nếu làm thủ công (Không dùng Interceptor):

1. **Khi xóa:** Bạn phải nhớ viết lệnh `Update` thuộc tính `IsDeleted = true` thay vì gọi hàm `_context.Remove()`.
    
2. **Khi đọc:** Mỗi lần viết câu lệnh lấy dữ liệu (LINQ), bạn _luôn luôn_ phải nhớ thêm điều kiện `.Where(x => !x.IsDeleted)`. Chỉ cần một Dev trong team quên thêm điều kiện này ở một câu API, hệ thống sẽ hiển thị cả những dữ liệu... đã xóa cho khách hàng xem.
    

##### Cách Interceptor và Global Query Filter giải quyết triệt để:

Sự kết hợp giữa `SaveChangesInterceptor` và `Global Query Filter` (Bộ lọc truy vấn toàn cục) tạo ra một vòng khép kín hoàn hảo:

##### Vế 1: Chặn đầu Ghi (Xử lý bằng Interceptor)

Khi lập trình viên gọi lệnh xóa tự nhiên: `_context.Orders.Remove(order);` Ngay tại hàm `SavingChanges`, Interceptor sẽ bắt được Entity này đang mang nhãn `Deleted`. Nó lập tức "tráo nhãn" và thay đổi giá trị:

C#

```CSharp
// Thay vì để EF tạo lệnh DELETE, ta ép nó quay xe thành lệnh UPDATE
entry.State = EntityState.Modified; // Đổi nhãn từ Deleted thành Modified
entry.CurrentValues["IsDeleted"] = true; // Gán cờ xóa thành true
```

Kết quả là EF Core bị "lừa" và sinh ra câu lệnh SQL: `UPDATE Orders SET IsDeleted = 1 WHERE Id = ...` thay vì lệnh `DELETE`.

##### Vế 2: Chặn đầu Đọc (Xử lý bằng Global Query Filter)

Để đảm bảo dữ liệu đã bị xóa mềm không bao giờ hiển thị lên giao diện, trong hàm `OnModelCreating` của `DbContext`, bạn cấu hình một bộ lọc mặc định:


```CSharp
modelBuilder.Entity<Order>().HasQueryFilter(o => !o.IsDeleted);
```

Từ lúc này, bất cứ khi nào bạn viết: `_context.Orders.ToList()`, EF Core sẽ tự động cài cắm thêm đoạn `WHERE IsDeleted = 0` vào câu lệnh SQL cuối cùng gửi xuống DB mà bạn không cần gõ một chữ `Where` nào cả.