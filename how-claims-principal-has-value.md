```CSharp
private string? Claim(string type) => _httpContextAccessor.HttpContext?.User.FindFirst(type)?.Value;
```

#### Luồng thực tế:
- Client gửi Request
```Http
POST /api/orders HTTP/1.1

Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
Content-Type: application/json

{
    "productId": 10
}
```

- Khi Authentication Middleware chạy
```CSharp
app.UseAuthentication();
```
Nó đọc
```CSharp
context.Request.Headers["Authorization"]
```
Và lấy được token đó đó: `eyJhbGciOiJIUzI1NiIs ...`
Giải mã Jwt -> Ở phần payload, lấy ra thông tin
```Json
{
    "sub": "123",
    "email": "hung@gmail.com",
    "role": "Admin"
}
```

Tạo: `ClaimsPrincipal`

Gán vào: `HttpContext.User` -> lúc này thằng đó có giá trị

#### HttpContext ?
Nó là toàn bộ ngữ cảnh (context) của một HTTP request trong ASP.NET Core. Nó bao gồm:
**Request**: mọi thứ client gửi lên (Method, Path. Query, Headers, Cookies, Body(**Stream**))
**Response:** mọi thứ server sẽ trả về (StatusCode, Headers, Body).
**Thông tin đã được ASP.NET Core bổ sung trong quá trình xử lí, như:**
- User: sau khi Authentication Middleware xác thực
- Items: để các middleware, filter, controller chia sẻ dữ liệu trong cùng request
- RequestServices để truy cập các service từ DI.
- Connect chứa thông tin kết nối (IP, cổng, ...)