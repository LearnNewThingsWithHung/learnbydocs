- **Data Annotations**: kiểu dùng truyền thống
```CSharp
public class LoginRequest  
{  
    [Required(ErrorMessage = "Email không được để trống")]  
    [EmailAddress(ErrorMessage = "Email không hợp lệ")]  
    public required string Email { get; set; }   
    [Required(ErrorMessage = "Mật khẩu không được để trống")]  
	public required string Password { get; set; } 
}
```

#### Fludent Validate là gì ?
**FluentValidation** là một thư viện mã nguồn mở cho .NET sử dụng **Fluent Interface** (gọi các hàm nối tiếp nhau bằng dấu chấm `.`) và **Lambda Expression** để xây dựng các quy tắc kiểm tra dữ liệu một cách mạnh mẽ, rõ ràng và tách biệt hoàn toàn khỏi class chứa dữ liệu (Model).

##### Tại sao nên dùng nó thay vì cách cũ (Data Annotations)?
- **Tách biệt mã nguồn (Separation of Concerns):** Bạn không cần "rải" các thuộc tính `[Required]`, `[EmailAddress]` làm bẩn Model/DTO nữa. Quy tắc validate nằm ở một class riêng.
- **Xử lý logic phức tạp:** Dễ dàng kiểm tra các điều kiện phụ thuộc (Ví dụ: _Nếu thuộc tính A là True thì thuộc tính B mới bắt buộc nhập_).
- **Dễ viết Unit Test:** Vì nó là các class riêng biệt, bạn có thể test logic kiểm tra dữ liệu cực kỳ nhanh chóng.
- **Hỗ trợ Localization tốt:** Dễ dàng tùy biến thông báo lỗi (Error Message) theo nhiều ngôn ngữ.

#### demo code
- Cấu trúc demo:
Ví dụ: trong request của folder Auth ở tầng service, giờ mình muốn validate những cái request cửa thằng Auth này, thì trong đó, mình sẽ tạo một folder riêng, và tạo các class để validate từng cái class nằm trong folder Auth

![[Store-Image/Pasted image 20260619221345.png]]
![[Store-Image/Pasted image 20260619221952.png]]

![[Store-Image/Pasted image 20260619221715.png]]

Ví dụ demo: Ta sẽ tạo một class request, kế thừa từ class AbtractValidator<> và bỏ vào cái class mà ta cần validate
Bên trong nó là một contructor rỗng, và
cứ mỗi một fields, thì chúng ta cứ validate riêng là được, và nhớ là cũng config lỗi của nó như ở file custom error luôn nhé, kiểu như này:

![[Store-Image/Pasted image 20260619222128.png]]

Kiểu vậy nè, rồi cứ nó mà ko fix với cái mà validate thì nó sẽ ném va middleware, nó bắt cái lỗi của mình là được


