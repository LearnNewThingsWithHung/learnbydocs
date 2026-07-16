
- Kĩ thuật **assembly**
```Csharp
public static class AssemblyReference  
{  
    public static readonly Assembly Assembly = typeof(AssemblyReference).Assembly;  
}
```

`Assembly` là gì ?

Giả sử ta có một cái project sau khi build tạo ra file: **HoaTheoMua.Service.dll**
Bên trong file .dll này có các class Service của mình
Thì toàn bộ thằng file .dll đó chính là một **Assembly**

`typeof(AssemblyReference);` là gì?
Thằng này có nghĩa là, cho tôi thông tin về class **AssemblyReference** (ở đây, class này là chính nó luôn `public static class AssemblyReference`, **có nghĩa là nó đang trỏ về chính nó, là toàn bộ cái project chứa nó luôn, tức HoaTheoMua.Service.dll luôn**)
Kết quả nhận được là một object kiểu `Type`
Ví dụ:
```CSharp
var type = typeof(AssemblyReference);  
Console.WriteLine(type.Name);

-> AssemblyReference
```

`typeof(AssemblyReference).Assembly` là gì ?
Bây giờ chính ta ráp 2 thằng lại với nhau:
`typeof(AssemblyReference)` : thông tin của class AssemblyReference này là gì
`.Assembly` : nó đang nằm ở file .dll nào
Ví dụ:
class **AssemblyReference** nằm trong file **HoaTheoMua.Service.dll**

- Kĩ thuật **Reflection**
`builder.Services.AddValidatorsFromAssembly(AssemblyReference.Assembly)` : mục hay đã tới
Dòng code này có nghĩa là: mình đưa cái **AssemblyReference.Assembly**(tức là cái file .dll chứa trông tin của thằng **AssemblyReference.Assembly**), và nhớ cho mình rằng là, file .dll là file build của toàn bộ project đang chứa cái thằng **AssemblyReference.Assembly** này, cho thằng **AddValidatorsFromAssembly**

Lúc này thằng **AddValidatorsFromAssembly** sẽ dùng một cái kĩ thuật gọi là `Reflection` nó sẽ ánh sạ lên cái project đó là lấy ra tất cả các class đang nằm trong đó.
Và bước tiếp theo, đối với mỗi thằng lấy ra, nó đều hỏi là, thằng đó có kế thừa từ **AbstractValidator**, nếu có thì thằng **AddValidatorsFromAssembly** sẽ tự động đăng kí vào DI chớ mình không cần phải làm kiểu như:

```CSharp
builder.Services.AddScoped<IValidator<UserDto>, UserValidator>(); builder.Services.AddScoped<IValidator<ProductDto>, ProductValidator>(); builder.Services.AddScoped<IValidator<OrderDto>, OrderValidator>(); ....
```


