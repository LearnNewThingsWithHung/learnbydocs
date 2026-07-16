
Các toán tử và cách sử dụng nó sao cho hợp lí
1. Dấu `?`
- Sử dụng khi giá trị mà mình muốn có thể null
Ví dụ: 
```CSharp
public string? Number {get; set}
public object? NhanhVnResponse {get; set;}
```
Khi mình ghi như thế này, thì compiler của .net là Rosyln, hiểu rằng là Number có thể null hoặc NhanhVnResponse, nếu vô tình NhanhVnResponse bị null mà mình chấm thuộc tính bên trong nó thì nó sẽ văng ra lỗi `NullReferenceException`
Ví dụ:
```CSharp
NhanhVnResposne? nhanhvnresponse = null. Ở đây giá trị của nhanhvnresponse bị null
Console.WriteLine(person.Name); -> crash chương trình tại đây, bị lỗi NullReferenceException
```
Còn nếu:
```CSharp
person?.Name	
```
Nó tương đương với:
```CSharp
person == null
    ? null
    : person.Name;
```
tránh crash, nếu null thì trả ra null để tránh crash, còn nếu
```CSharp
person!.Name	
```
- Lỡ mà nó trả ra null một phát thì bị lỗi `NullReferenceException`
2. Dấu `!`
- Sử dụng khi giá trị mình biết là sẽ không null thì mình thêm dấu `!` để báo cho compiler biết rằng à, chỗ đó sẽ trả ra giá trị đừng cảnh báo cho mình nữa
```CSharp
public string? Number {get; set;} = null!;
```
- Giả sử, mình khai báo trong request mà FE gửi xuống có một field như thế này, và có thể null, nếu không có `!`(FE có thể truyền xuống hoặc không cũng được) nhưng khi build sẽ bị lỗi cảnh báo warning.

3. Dấu `??`: toán tử xử lý null thật sự
- Giả sử:
```CSharp
public string? name = null;
public string result = name ?? "Unknown";
```
- Lúc này compiler sẽ hiểu tương tự như là:
```CSharp
if (name != null)
    result = name;
else
    result = "Unknown";
```
- Nếu name mà null thì thay bằng "Unknown" -> ứng dụng tránh chương trình bị crash

