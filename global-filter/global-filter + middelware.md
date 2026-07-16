#### Vấn đề gặp phải, lúc bấy giờ:
Ví dụ thực tiễn, trong Service Order, có tới 4 hàm, nhưng hàm nào hiện tại cũng đều có validate hết.

**Sự lặp lại:** Mỗi method trong class service đó, ở contructor ta đều phải khai báo validate cho nó, đối với trong class này, ta khai báo tới 4 cái validate cho các method bên trong -> contructor bị phòng to
chưa kể, mỗi cái method trong class này, ta đều phải, lặp lại một đoạn code kinh điển mà chúng theo một format chung đó là: gọi validate(request), bắt lỗi và ném cho middleware. Điều gì xảy ra nếu trong class đó có tới 100 hàm con với mỗi hàm có một validate riêng. Vậy phải khai báo 100 cái validate vào trong constructor, lặp lại 100 cái validate trong service.

-Ví dụ: đây mới chỉ là một phần nhỏ validate trong Service Authen
```CSharp
public Service(AppDbContext context,   
	IValidator<Request.RegisterRequest> registerValidator,   
	IValidator<Request.LoginRequest> loginValidator,  
    IValidator<Request.ForgotPasswordRequest> forgotPasswordValidator,  
    IValidator<Request.ResendVerifyEmailRequest> resendVerifyEmailValidator,  
    MailService.IService emailService, CacheService.IService cacheService,   
JwtService.IService jwtService, IHttpContextAccessor httpContextAccessor)  
{  
    _dbContext = context;  
    _registerValidator = registerValidator;  
    _forgotPasswordValidator =  forgotPasswordValidator;  
    _loginValidator = loginValidator;  
    _resendVerifyEmailValidator = resendVerifyEmailValidator;  
    _emailService = emailService;  
    _cacheService = cacheService;  
    _jwtService = jwtService;  
    _httpContextAccessor = httpContextAccessor;  
}
```
-Ở trong function, ta cũng lặp lại một đoạn code tương tự:
```CSharp
var validationResult = await _registerValidator.ValidateAsync(request);  
  
if (!validationResult.IsValid)  
{  
    throw new ValidationExceptionBuid("Invalid registration request");  
}

var validationResult = await _loginValidator.ValidateAsync(request);  
  
if (!validationResult.IsValid)  
{  
    throw new ValidationExceptionBuid("Invalid login request");  
}
...
```

#### Global Filter Validate
Để giải phóng cho tầng service, người ta nghĩ ra **Global Filter Validate**

Lúc ban đầu: Bạn hãy tưởng tượng như thế này, lúc bình thường(code cũ), khi FE gọi một cái API qua tầng controller, controller gọi hàm ở service, khi request đi vào service, điều đầu tiên là validate cái request mà FE gửi cho BE, xem thử nó có hợp lệ không rồi mới cho đi tiếp, request/API khác cũng tương tự như vậy, nếu có lỗi thì ném ra, và Middleware sẽ bắt nó lại, và trả ra cho FE.

Khi áp dụng global filter validate thì, dường như mọi thứ thay đổi khá nhiều
Lúc này, thay vì mỗi request vào trong service rồi mới validate tương ứng với các hàm/api mà FE gọi xuống. 

Thì bây giờ, global filter đóng vai trò như một người bảo vệ, chúng ta không cần phải khai báo từng cái IValidate trong contructor hay lặp lại code gọi từng cái hàm validate cho từng cái request bên trong  các hàm nữa. Mà global filter đóng vai trò như một người bảo vệ đứng canh cổng, trước khi request đó được vào controller luôn. Nếu validate hợp lệ, thì bác bảo vệ này cho request đi vào controller -> Service. Nếu dữ liệu lỗi, thì bác chặn lại luôn.

À, lúc này ta khá là mường tượng được vai trò của nó rồi:

Lúc trước, ta phải validate trong từng cái hàm, nếu invalid request thì tầng service ném lỗi, và middleware sẽ bắt nó lại.
Còn bây giờ, global filter chặn nó trước khi nó vào được controller luôn.

#### Cách Filter Validate hoạt động
Filter global hoạt động theo một nguyên lí rất hay trong lập trình gọi là AOP(Aspect-Oriented Programming - Lập trình hướng khía cạnh). Nó hoạt động theo 3 bước tuần tự sau:

**Bước 1:** Đăng ký tự động
Bóc tách nghĩa: global filter =  toàn cầu + filter -> filter toàn bộ
Khi ứng dụng khởi động, bạn cấu hình để hệ thống có thể tự động quét hết tất cả các class  validate. Lúc này hệ thống sẽ tự hiểu là: nếu gặp Request dạng CreateOrderRequest thì dùng CreateOrderValidator để kiểm tra nhé.

**Bước 2**: Intercept
Khi request đi qua Global Filter -> Global Filter tự động nhìn xem request này thuộc class nào, và nó sẽ tự động tìm xem có class validate nào mà validate có request này không. Nếu tìm thấy, nó sẽ tự gọi hàm validate tương ứng đó ra.

**Bước 3:** Đưa ra phán quyết.
Nếu request hợp lệ thì cho đi tiếp
Nếu request fail: Filter tự động ngắt nguồn xử lý. Tự động map các lỗi thành một định dạng chung và trả về cho Client

Lúc này chúng ta, không cần phải inject `IValidator` nữa. Và cũng không còn phải bị lặp lại đoạn code `if (!result.IsValid) .. throw ...` ở bất kì đâu nữa. Mà Global Filter sẽ làm chuyện đó cho chúng ta. 

#### Global filter giúp ta gọi các hàm validate, còn middleware cũng giúp chúng ta validate. Vậy giúp khác nhau như thế nào?

Đó là: "Khả năng nhận biết **Model Binding**"

##### Đầu tiên, chúng ta cần hiểu rõ về pipeline của một luồng request nhé.
HTTP Request -> **Middleware** -> Routing -> **Model Binding** -> **Action Filter** -> ControllerAction -> Response -> Middleware
(Chú ý những chỗ mà mình bold nhé)

**Model Binding** là gì?
	Là quá trình ASP.NET Core tự động lấy dữ liệu từ HTTP Request và chuyển thành object C#
	
Ví dụ: Client gửi: 
```http
POST /api/orders
Content-Type: application/json
{
    "productId": 10,
    "quantity": 2,
    "address": "HCM"
}
```

Controller:
```CSharp
[HttpPost]
public IActionResult CreateOrder(CreateOrderRequest request)
{
    ...
}
```

với CreateOrderRequest:
```CSharp
public class CreateOrderRequest
{
    public int ProductId { get; set; }
    public int Quantity { get; set; }
    public string Address { get; set; }
}
```
 
 Nếu để ý, bạn sẽ thấy, FE gửi body ở dạng json, nhưng trong controller lại nhận CreateOrderRequest, nhưng trong khi đó, ở controller không có dòng code giúp parse cái json đó thành object CreateOrderRequest kiểu như thế này:
 
 ```CSharp
 JsonSerializer.Deserialize<CreateOrderRequest>(...)
 ```

**Câu hỏi đặt ra**: Ai đã âm thầm giúp ta làm chuyện đó ? -> Chính là: **Model binding(Stream -> Object)**

##### Vậy sau khi qua Model binding, thì lúc này body của Http Request đang ở dạng object rồi, vậy trước đó ở tầng Middleware thì dữ liệu ở dạng gì ?
-> Đó là Stream. 
Stream là gì ? Dịch ra nghĩa tiếng việt nó là dòng chảy, ở đây chúng ta hình dung nó là một **đường ống/ ống nước**.
**Ống nước** thì phải có nước chảy bên trong ống. Vậy nước là gì: **Bytes** 

Dễ hình dung hơn, khi FE gửi xuống một Http Request, phần body chúng ta thấy ở dạng json, nhưng máy tính ko hiểu, nên chúng convert thành Bytes để hiểu. 

-> Như chúng ta thấy, khi Http Request đi vô, thì Middleware nằm ở trước Model binding, nên nó chỉ có thể validate được Request Line và Headers của Http Context thui (giải thích thêm: vì nó chỉ thấy body của Http Request khi đi vô chỉ ở dạng stream thôi)

==Cũng chính vì vậy, mà mình đặt Global Filter mà mình cấu hình ở Action Filter chớ ko đặt được ở Middleware được.==

Validation thì  có 3 tầng:
HTTP Validation: Middleware
Model Validation (Model Binding xong)
Bussiness Validation (Service)

