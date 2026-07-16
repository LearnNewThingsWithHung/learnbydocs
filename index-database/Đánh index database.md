## 1. Thiết lập Model & Setup trong Entity Framework Core

Đầu tiên, đây là cách em định nghĩa Entity trong C#. Anh sẽ chỉ cho em cú pháp của cả hai trường hợp: **Không Index** và **Có Index**.

### Case 1: Không có Index (Mặc định)

C#

```CSharp
public class User
{
    public long Id { get; set; } // PK -> Mặc định DB tự tạo Clustered Index cho cột này
    public string Name { get; set; }
    public string Email { get; set; } // Cột này CHƯA có Index
    public DateTime CreatedAt { get; set; }
}
```

### Case 2: Cấu hình Có Index bằng Fluent API

Để bảo EF Core tạo Index cho cột `Email` khi Migration, em vào file `AppDbContext.cs` và cấu hình trong hàm `OnModelCreating`:

C#

```CSharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<User>(entity =>
    {
        // Đánh Index cho cột Email, và ép nó là Unique (vì Email không được trùng)
        entity.HasIndex(u => u.Email)
              .IsUnique(); 
    });
}
```

> **Hậu quả nếu quên `IsUnique()` ở App thực tế:** Nếu không có `IsUnique`, DB vẫn cho phép chèn 2 Email trùng nhau. Khi User bấm "Quên mật khẩu", hệ thống bắn OTP về, DB trả về 2 dòng => Code Backend của em dùng lệnh `.SingleOrDefault()` sẽ bị **Crash (Nổ exception)** ngay lập tức vì trả về nhiều hơn 1 kết quả.

## 2. Luồng Code Backend (API Controller / Service)

Dù em có đánh Index hay không, thì **Code C# của em KHÔNG HỀ THAY ĐỔI**. Em vẫn viết một câu lệnh LINQ quen thuộc như thế này:

C#

```CSharp
public async Task<User?> GetUserByEmailAsync(string email)
{
    // Đoạn code LINQ này dùng chung cho cả 2 trường hợp
    return await _context.Users
                         .FirstOrDefaultAsync(u => u.Email == email);
}
```

### Vậy sự khác biệt nằm ở đâu?

Nó nằm ở cách **EF Core biên dịch ra SQL** và cách **Database Server thực thi câu SQL đó**.

## 3. So sánh dưới "Góc nhìn của Database"

Khi API gọi hàm trên với tham số `email = "target_user@gmail.com"`, EF Core sẽ sinh ra câu lệnh SQL như sau:

SQL

```SQL
SELECT TOP(1) [u].[Id], [u].[Name], [u].[Email], [u].[CreatedAt]
FROM [Users] AS [u]
WHERE [u].[Email] = N'target_user@gmail.com'
```

Hãy xem sự khác biệt kinh hoàng giữa 2 trường hợp khi DB nhận câu SQL này:

### Trường hợp 1: KHÔNG CÓ INDEX

- **Hành động của DB:** Nó không biết thằng `target_user@gmail.com` nằm ở đâu cả. Nó bắt buộc phải làm một việc gọi là **Table Scan** (hoặc _Clustered Index Scan_). Nó mở ổ cứng ra, đọc dòng số 1, không phải -> đọc dòng số 2, không phải... đọc cho tới khi tìm thấy (hoặc đọc hết cả 1 triệu dòng nếu email đó không tồn tại).
    
- **Báo cáo tài nguyên (Execution Plan):**
    
    - **CPU:** Nhảy vọt lên cao vì phải so sánh chuỗi 1 triệu lần.
        
    - **I/O (Đọc đĩa):** Cực kỳ nặng vì phải nạp toàn bộ 1 triệu dòng từ ổ cứng lên RAM.
        
    - **Thời gian chạy (Duration):** Khoảng **200ms - 500ms** cho 1 request. (Nếu có 100 người cùng sreach một lúc => Server DB nghẽn mạch, API đứng hình).
        


### Trường hợp 2: CÓ INDEX TRÊN CỘT EMAIL
[[“Nếu cái nào cũng đánh dấu, thì lúc tìm chẳng phải vẫn phải lướt qua từng cái đánh dấu đó sao”]]
- **Hành động của DB:** Nó nhìn vào cây Index (B-Tree) của cột `Email`. Do cây Index đã được sắp xếp sẵn theo thứ tự chữ cái, DB chỉ mất khoảng **3 đến 5 phép toán so sánh** (Index Seek) là tìm ra ngay vị trí chính xác của dòng dữ liệu trên ổ cứng.
    
- **Báo cáo tài nguyên (Execution Plan):**
    
    - **CPU:** Gần như bằng 0%.
        
    - **I/O (Đọc đĩa):** Chỉ đọc đúng vài KB dữ liệu của cây Index và 1 dòng duy nhất của bảng chính.
        
    - **Thời gian chạy (Duration):** **< 1ms (0.000 giây)**. Tốc độ bàn thờ!
## 5. Bài học thực chiến cho Backend Engineer (C# / EF Core)

1. **Đừng tin blindly vào ORM:** EF Core rất mạnh, nhưng nó chỉ là thằng "dịch tiếng C# sang tiếng SQL". Nếu em thiết kế DB ngu (thiếu index), EF Core vẫn dịch ra SQL chạy được, nhưng hệ thống sẽ sập khi Data lớn. Luôn phải check xem DB của mình đã có Index cho các cột nằm trong hàm `.Where()`, `.FirstOrDefault()`, `.Join()` chưa.
    
2. **Hậu quả của việc lạm dụng:** Nếu em thấy Index ngon quá, em tiện tay gắn Index cho mọi cột (`Name`, `Age`, `Address`, `CreatedAt`...). Lúc này, hàm `_context.Users.AddAsync(newUser)` của em sẽ chạy **chậm như rùa bò** vì DB phải cập nhật mười mấy cây Index cùng một lúc.

