### Local storage là gì ?
Local Storage là bộ nhớ lưu trữ dữ liệu lâu dài trên trình duyệt, cho phép JavaScript đọc và ghi dữ liệu dưới dạng key-value. Thường được dùng để lưu JWT hoặc thông tin người dùng, nhưng có rủi ro bảo mật vì JavaScript có thể truy cập được.

Vì nó có thể cho JS đọc và ghi được dữ liệu, thì mới nguy hiểm. Vì code JS có thể đọc và lấy dữ liệu từ localStorage, nếu hacker có thể cấy cái code JS chứa mã độc vào trình duyệt của chúng ta thì chúng có thể lấy dữ liệu từ localStorage
### HttpOnly cookie là gì?
HttpOnly Cookie cũng lưu dữ liệu trên trình duyệt nhưng JavaScript không thể truy cập được. Browser sẽ tự động gửi cookie lên server trong mỗi request, giúp giảm nguy cơ bị đánh cắp token qua XSS nên thường được ưu tiên hơn Local Storage trong các hệ thống yêu cầu bảo mật cao.

#### 2 lỗ hỏng lớn:
1. **XSS**: khi một thằng hacker cấy JS chứa mã độc và web của mình, nếu mình mình lưu refresh token của localStorage thì toang, JS đọc được localStorage rồi gửi API chứa cái refresh token lên cho nó, nó cầm refresh token đó là login vào một server khác .
- **Local Storage (Sợ XSS):** Hacker phải **đọc được** token, copy về máy hacker, rồi hacker tự gửi request từ máy của nó.
1. **CSRF**: thì được cái là Js không được HttpOnly Cookia, nhưng nó có một cái lỗ hỏng đó là bất cứ cái request nào nó cũng gửi cái cookie vô hết. 
   - **HttpOnly Cookie (Sợ CSRF):** Hacker **không đọc được** cookie, nó phải đứng từ xa điều khiển trình duyệt của nạn nhân tự gửi request đi.  
   - Để khắc phục điều đó, các hệ thống lớn người ta dùng một thuộc tính tên là **SameSite** tức là chỉ cho phép tự động gửi cookie này khi user đang đứng trực tiếp trên web của nó.

| **Tiêu chí**         | **Same-Origin Policy (SOP) & CORS**                                            | **Thuộc tính SameSite của Cookie**                                                                    |
| -------------------- | ------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| **Đối tượng bảo vệ** | **Dữ liệu/Tài nguyên (Response Data)** từ API.                                 | **Danh tính/Phiên đăng nhập (Session/Cookie)** của người dùng.                                        |
| **Nơi thực thi**     | Trình duyệt chặn không cho **đọc dữ liệu** trả về.                             | Trình duyệt chặn không cho **gửi Cookie đi** kèm request.                                             |
| **Câu hỏi cốt lõi**  | "Trang web độc hại có được quyền **xem** kết quả trả về từ API của tôi không?" | "Request từ trang web khác bay tới có được phép **đính kèm** chứng minh thư (Cookie) của user không?" |
[[sop & cors]]
