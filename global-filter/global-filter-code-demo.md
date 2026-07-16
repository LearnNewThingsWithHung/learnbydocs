
Đăng kí global filter:

```CSharp
builder.Services.AddControllers(options =>
{
    // Đăng ký ValidationActionFilter làm Global Filter.
    // Mọi request đi vào Controller đều sẽ chạy qua filter này trước.
    options.Filters.Add<ValidationActionFilter>();
})
.AddJsonOptions(options =>
{
    // Tránh lỗi khi serialize object có circular reference (tham chiếu vòng).
    //
    // Ví dụ:
    //
    // Order
    //   └── OrderDetails
    //          └── Order
    //
    // Nếu serializer duyệt:
    // Order
    //   -> OrderDetails
    //        -> Order
    //             -> OrderDetails
    //                  -> ...
    //
    // sẽ tạo thành vòng lặp vô hạn và gây exception.
    //
    // ReferenceHandler.IgnoreCycles sẽ bỏ qua những object đã được serialize trước đó,
    // giúp tránh lỗi circular reference khi trả JSON về client.
    options.JsonSerializerOptions.ReferenceHandler =
        System.Text.Json.Serialization.ReferenceHandler.IgnoreCycles;
});
```

```CSharp
public class ValidationActionFilter : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,     // Chứa thông tin của request hiện tại (HttpContext, ActionArguments, RouteData, ...)
        ActionExecutionDelegate next)       // Gọi next() để tiếp tục thực thi Action
    {
        // Lấy IServiceProvider của request hiện tại để resolve các service đã đăng ký trong DI
        var services = context.HttpContext.RequestServices;

        // Danh sách lưu tất cả lỗi validation
        var errors = new List<ValidationError>();

        // ActionArguments chứa toàn bộ tham số của Action sau khi Model Binding hoàn thành
        // Ví dụ:
        // public IActionResult Create(CreateOrderRequest request, int shopId)
        // => ActionArguments sẽ chứa:
        //    request (CreateOrderRequest)
        //    shopId (int)
        foreach (var argument in context.ActionArguments.Values)
        {
            // Không có object để validate thì bỏ qua
            if (argument is null)
            {
                continue;
            }

            // Tìm validator tương ứng với kiểu của argument trong DI.
            // Chỉ tìm các service có kiểu IValidator<T>.
            //
            // Ví dụ:
            // builder.Services.AddScoped<IValidator<CreateOrderRequest>, CreateOrderRequestValidator>();
            //
            // Nếu argument là CreateOrderRequest
            // => Resolve ra CreateOrderRequestValidator.
            //
            // Các service khác như:
            // IOrderService
            // DbContext
            // ILogger
            // ... sẽ không được lấy.
            var validator = ResolveValidator(services, argument.GetType());

            // Không đăng ký validator thì bỏ qua
            if (validator is null)
            {
                continue;
            }

            // Tạo ValidationContext để FluentValidation thực hiện validate
            var validationContext = new ValidationContext<object>(argument);

            // Thực hiện validate
            var result = await validator.ValidateAsync(
                validationContext,
                context.HttpContext.RequestAborted);

            // Nếu có lỗi thì map sang ValidationError của hệ thống
            if (!result.IsValid)
            {
                errors.AddRange(result.Errors.Select(e => new ValidationError
                {
                    Field = ToCamelCasePath(e.PropertyName),
                    Message = e.ErrorMessage
                }));
            }
        }

        // Nếu tồn tại lỗi validation thì throw exception.
        // Controller sẽ không được thực thi.
        if (errors.Count > 0)
        {
            throw new ValidationExceptionErrorV1(errors);
        }

        // Không có lỗi => tiếp tục thực thi Controller Action
        await next();
    }

    // Chuyển PropertyName về camelCase.
    // Ví dụ:
    // Items[0].Quantity
    // =>
    // items[0].quantity
    private static string ToCamelCasePath(string propertyName)
    {
        if (string.IsNullOrEmpty(propertyName))
        {
            return propertyName;
        }

        var segments = propertyName.Split('.');

        for (var i = 0; i < segments.Length; i++)
        {
            segments[i] = CamelCaseSegment(segments[i]);
        }

        return string.Join('.', segments);
    }

    private static string CamelCaseSegment(string segment)
    {
        if (string.IsNullOrEmpty(segment) || !char.IsUpper(segment[0]))
        {
            return segment;
        }

        return char.ToLowerInvariant(segment[0]) + segment[1..];
    }

    // Resolve validator theo kiểu của request.
    //
    // Đồng thời hỗ trợ kế thừa.
    //
    // Ví dụ:
    //
    // class CreateRequest
    // class UpdateRequest : CreateRequest
    //
    // Chỉ đăng ký:
    // IValidator<CreateRequest>
    //
    // Khi validate UpdateRequest:
    // UpdateRequest
    //      ↓
    // CreateRequest
    //      ↓
    // object
    //
    // Nếu tìm thấy validator ở lớp cha thì vẫn sử dụng được.
    private static IValidator? ResolveValidator(
        IServiceProvider services,
        Type argumentType)
    {
        for (
            var type = argumentType;
            type is not null && type != typeof(object);
            type = type.BaseType)
        {
            var validatorType = typeof(IValidator<>).MakeGenericType(type);

            if (services.GetService(validatorType) is IValidator validator)
            {
                return validator;
            }
        }

        return null;
    }
}
```