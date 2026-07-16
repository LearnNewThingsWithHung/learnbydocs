#### Custom Error: GlobalExceptionHandlerMiddleware
Lúc trước khi học a Tân, chúng ta sẽ dùng các exception mà .Net hỗ trợ, chỗ nào lỗi, chúng ta cứ throw là thằng `GlobalExceptionHandlerMiddleware` sẽ bắt. Nếu chúng ta để ý, có một vài lỗi mà .Net hỗ trợ, hoặc các extenstion khác có. Nhưng có một vài lỗi thì không. 

Bây giờ, hệ thống đang lớn dần lên, chúng ta cần **ném** ra những lỗi chính xác(để FE có thể bắt lỗi đó và setup tự gọi hàm, ví dụ: khi hết accesstoken mình nằm ra `status: 401`, nhưng details:`"MISSING_ACCESS_TOKEN"` hoặc `status: 401`, details `EXPIRED_ACCESS_TOKEN` )

Tiến hành đi custom nó,  búi tóc lên, ta quyết tâm refactor code lại:

```CSharp 
public class GlobalExceptionHandlerMiddleware : IMiddleware  
{  
    private readonly IHostEnvironment _environment;  
    public GlobalExceptionHandlerMiddleware(  
        IHostEnvironment environment,  
        ILogger<GlobalExceptionHandlerMiddleware> logger)  
    {        
	    _environment = environment;  
        _logger = logger;  
    }  
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)  
    {        
	    try  
        {  
            await next(context);//cho request đi tiếp   
		}  
        catch (Exception ex)  
        {  
            _logger.LogError(ex, "Unhandled exception occurred while processing request {Path}",  context.Request.Path);  
  
            if (context.Response.HasStarted)  
            {                // Lỗi này lỗi nâng cao  
                _logger.LogWarning("The response has already started, the global exception middleware will not write an error response");  
                throw;  
            }  
            var statusCode = MapStatusCode(ex);  
            context.Response.ContentType = "application/json"; 
              

            var response = ApiResponseFactory.ErrorResponse(  
                message: ResolveClientMessage(ex, statusCode),  
                errors: _environment.IsDevelopment() ? new { detail = ex.Message } : null,     
                traceId: context.TraceIdentifier);   
  
            await context.Response.WriteAsJsonAsync(response);  
        }  
    }  
    private static int MapStatusCode(Exception ex)  
    {        return ex switch 
{  
            ArgumentException => StatusCodes.Status400BadRequest, 
            InvalidOperationException => StatusCodes.Status400BadRequest 
            UnauthorizedAccessException => StatusCodes.Status401Unauthorized,  
            KeyNotFoundException => StatusCodes.Status404NotFound, 
            _ => StatusCodes.Status500InternalServerError 
        };  
    }  
    private static string ResolveClientMessage(Exception ex, int statusCode)  
    {        return statusCode >= 500 ? "An unexpected error occurred" : ex.Message;  
    }}
```

- Khi mình quăng lỗi, thì thằng middleware này thì bắt lại và dựa vào các lỗi, và status mình config và map nó theo cái thằng này:
```CSharp
var response = ApiResponseFactory.ErrorResponse(  
                message: ResolveClientMessage(ex, statusCode),  
                errors: _environment.IsDevelopment() ? new { detail = ex.Message } : null,     
                traceId: context.TraceIdentifier);   
```

```CSharp
private static int MapStatusCode(Exception ex)  
    {        return ex switch 
{  
            ArgumentException => StatusCodes.Status400BadRequest, 
            InvalidOperationException => StatusCodes.Status400BadRequest 
            UnauthorizedAccessException => StatusCodes.Status401Unauthorized,  
            KeyNotFoundException => StatusCodes.Status404NotFound, 
            _ => StatusCodes.Status500InternalServerError 
        };  
    }  
```

- Đoạn code trên có thể cho thấy được những lỗi có có trong này nó default là: 
` _ => StatusCodes.Status500InternalServerError `


#### Tiến hành độ lại:
```CSharp
public class GlobalExceptionHandlerMiddleware : IMiddleware  
{  
    private readonly IHostEnvironment _environment;  
    private readonly ILogger<GlobalExceptionHandlerMiddleware> _logger;  
  
    public GlobalExceptionHandlerMiddleware(  
        IHostEnvironment environment,  
        ILogger<GlobalExceptionHandlerMiddleware> logger)  
    {        _environment = environment;  
        _logger = logger;  
    }  
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)  
    {        try  
        {  
            await next(context);  
        }        catch (Exception ex)  
        {            _logger.LogError(ex, "Unhandled exception occurred while processing request {Path}",   
                            context.Request.Path);  
  
            if (context.Response.HasStarted)  
            {                _logger.LogWarning("The response has already started, the global exception middleware will not write an error response");  
                throw;  
            }                          
//context.Response.Clear();  
            context.Response.ContentType = "application/json";  
            AppException appEx = ex switch  
            {  
                AppException alreadyAppEx => alreadyAppEx,  
                _ => new ServerException("An unexpected system error occurred.")   
            };  
                        context.Response.StatusCode = appEx.StatusCode;  
            var response = ApiResponseFactory.Error(  
                title: appEx.Title,  
                status: appEx.StatusCode,  
                detail: appEx.StatusCode >= 500 ? "An unexpected error occurred." : ex.Message,  
                messageCode: appEx.MessageCode,  
                errors: _environment.IsDevelopment() ? new { detail = ex.Message } : null,   
                traceId: context.TraceIdentifier  
            );  
            await context.Response.WriteAsJsonAsync(response);  
        }    }}
```

- Về bản chất, đoạn code trên tương tự như đoạn code dưới thôi
` var statusCode = MapStatusCode(ex);  ` : đoạn code trên thì bắt lỗi ở bước này, rồi map sang `ApiResponseFactory`, cho nó custom lỗi cho đẹp

- Ở đoạn code dưới: 
```CSharp
AppException appEx = ex switch  
            {  
                AppException alreadyAppEx => alreadyAppEx,  
                _ => new ServerException("An unexpected system error occurred.")   
            };
```
thì đoạn này bắt lỗi, thì những cái mà trong `AppException` sẽ do mình custom bởi chính mình, còn nếu ko có trong cái mà mình custom, thì lấy default là `_ => new ServerException("An unexpected system error occurred.")`

Vậy bây giờ là sẽ biến hành đi custom, những lỗi mà ta muốn, ta muốn cái lỗi có những gì thì ta sẽ đi custom những cái mà tôi muốn, ở đây mình đi custom thằng `AppException` này nhé.

Trong layer Service, mình sẽ đi tạo một folder Exceptions,
sau đó tạo abtractions, mục đích làm base, sau đó các thằng có lỗi khác nhau thì tạo các class riêng kế thừa từ thằng base, nhưng vẫn giữ được cái base chung đó
```CSharp
public abstract class AppException: Exception  //thằng này kế từ thằng ex
{  
    public string Title { get; set; }   
public int StatusCode { get; set; }  
    public string MessageCode { get; set; }  
    protected AppException(  
        string title, int statusCode, string messageCode,  string detail): base(detail)  
    {        Title = title;  
        StatusCode = statusCode;  
        MessageCode = messageCode;  
    }  
}
```

- vậy các thằng sau kế thừa từ thằng exception của mình sẽ có 3 thuộc tính là gì
`Title`: Lỗi gì
`StatusCode`: Mã lỗi
`MessageCode`: Tin nhắn lỗi

- Demo
```CSharp
public class MissingAccessTokenException : AppException  
{  
    public MissingAccessTokenException()  
        : base("Unauthorized", 401, "MISSING_ACCESS_TOKEN",   
"Access token is missing.") { }  
}  
  
public class ExpiredAccessTokenException : AppException  
{  
    public ExpiredAccessTokenException()  
        : base("Unauthorized", 401, "EXPIRED_ACCESS_TOKEN",   
"Access token has expired.") { }  
}  
  
public class ExpiredRefreshTokenException : AppException  
{  
    public ExpiredRefreshTokenException()  
        : base("Unauthorized",401,  "EXPIRED_REFRESH_TOKEN",   
"Refresh token has expired. Please login again.") { }  
}
```

cái này là em fix cứng chỉ cần ném lỗi là ra

```CSharp
public class ForbiddenException : AppException  
{  
    public ForbiddenException(string detail)  
        : base("Forbidden", 403, "FORBIDDEN", detail) { }  
}
```
đó đó, đối với cái này ném throw ra là rồi bỏ details vào là đúng bài luôn

- Nhược điểm: 
	- vì mình tự custom nên tự tay setup nhiều lỗi -> hơi bất tiện
	- Và vì mình bọc bằng lỗi của mình, nên nối nó ko có lỗi mà mình tự tay custom nên mặc định là 500 -> điều này khác bất tiện
- Vậy các hệ thống lớn họ tự tay custom hết à kkkk ....