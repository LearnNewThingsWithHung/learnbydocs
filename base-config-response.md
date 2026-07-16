 Tại sao ta lại cần config api response này:
 - Để thống nhất format chung cho tất cả mọi người tri trả ra kết quả
 - Config giống với FE
 - Clean, tất cả trong một folder, không nằm rải rác

Phân tích: Giả sử FE, đưa cho mình cái response như thế này, và bảo là BE hãy custom theo và dùng đi, cho chung format, thì mình tiến hình code thôi
![[Store-Image/Pasted image 20260620105614.png]]
Thì chúng ta tiến hành config y chang như vậy thôi,  rồi sau đó cứ gọi ra là được\
Tạo một folder model, bên trong response, rồi custom theo tôi

```CSharp
public abstract class ApiResponse  
{  
    public bool IsSuccess { get; set; }  
    public bool IsFailed { get; set; }  
    public object? Error { get; set; }  
    public string? TraceId { get; set; }  
    public DateTime TimestampUtc { get; set; } }  
  
public class BaseResponse : ApiResponse  
{  
    public object? Value { get; set; }  
}
```

- Cách làm:
Tại một static class có tên là **ApiResponseFactory** bên trong nó chứa object **BaseResponse** này, nhận vào các thông số như bên dưới, thì lúc này object? data, tức là object value , rồi rap lại với cái thằng api response đó

```CSharp
public static class ApiResponseFactory  
{  
    public static BaseResponse Base(  
        object? data, bool isSuccess = true,   
object? errorCustom = null, string? traceId = null)  
    {        return new BaseResponse  
        {  
            IsSuccess = isSuccess,  
            IsFailed = !isSuccess,  
            Value = isSuccess ? data : null,   
Error = !isSuccess ? errorCustom : null,   
TraceId = traceId,  
            TimestampUtc = DateTime.UtcNow  
        };  
    }
```

Vậy cứ mỗi lần dùng ở controller thì cứ:
**ApiResponseFactory**.Base(....) là được.