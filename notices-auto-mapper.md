1. **Tình huống:** Auto mapper là gì ? và tại sao nên dùng nó.

Ví dụ ta đang có một bảng User, bảng này có các fields sau:

```CSharp
public class User
{
	public string Name {get; set;}
	public string Password {get; set;}
	public string Email {get; set;}
	public string DateOfBirth {get; set;}	
}
```

Ta nhận được 1 api từ FE, đó là update hết toàn bộ các fields trong User. Lúc này ta nghĩ rằng, trong code, tôi chỉ cần kiểu như:

```CSharp
if(request.Name != null)
{
	user.Name = request.Name;
}
if(request.Password != null)
{
	user.Password = request.Password;
}
if(request.Email != null)
{
	user.Email = request.Email;
}
if(request.DateOfBirth != null)
{
	user.DateOfBirth = request.DateOfBirth;
}
```

Lúc này, ta cảm thấy nó dễ. Nhưng, lỡ một cái table có tới 1000 cái fields, dẫy là một cũng phải dùng 1000 cái if else hả ??? . Vậy thì đâu có được.

2. **Giải pháp**: 
Thực ra sẽ có nhiều thằng giúp chúng ta làm việc map đó, nhưng ở đây Hùng sẽ dùng AutoMapper từ thư viện: **AutoMapper**

**Hiểu nôm na**: chúng ta sẽ cấu hình, cho thằng AutoMapper này nhận biết. Những cái gì nó khác null thì nó sẽ update.

Cụ thể:
```CSharp
public class ProductMapping: Profile  
{  
    public  ProductMapping()  
    {        CreateMap<Request.UpdateProductRequest, Repository.Entity.Product>()  
            .ForAllMembers(opts => opts.Condition(  
                (src, dest, srcMember) => srcMember != null));  
    }
}
```

Cụ thể mình sẽ cấu hình thằng này, nó sẽ nhận vào: **Request.UpdateProductRequest** là **src** và map vào **Repository.Entity.Product** là **dest**, với các **properties** là **srcMember**, ở đây mình cấu hình **srcNember** khác null.

**Cụ thể flow:** map sẽ duyệt qua từng cái **srcNumber**(properties) ở trong class **src**(Request.UpdateProductRequest) nếu nó khác **null**(tức properties != null), thì sẽ tiến hành update trong **dest**(Repository.Entity.Product)

=> Đó chỉ có vậy thôi mà tiện lắm. Nhưng bù lại, nó có một cái bất tiện
Như mình nói, nếu khác null thì fields đó sẽ tự update. Nhưng lỡ người dùng muốn update fields đó thành **null thì sao ?** Đó là một vấn đề khác nữa, tự **research** thử nha.

Một vấn đề cần lưu ý, về tối ưu hiệu năng, anh **Tân nói**:
"Thường ko nên dùng automapper ở api get ,hay bị lỗi là lấy 100 thằng lên, rồi duyệt 100 thằng -> tạo ra một kiểu mới, thì nó ko tối ưu".

**Cụ thể:** **Một ví dụ hay về các api Get**

Ta có một cái **ProductResponse** là: 
```CSharp
public class ProductResponse
{
    public Guid Id { get; set; }
    public string? Name { get; set; }
    public decimal Price { get; set; }
    public string? CoverImage { get; set; }
}
```

- Một hàm giúp tôi map từ GetProduct ra List`<ProductResponse>` thoi.

```CSharp
public async Task<List<ProductResponse>> GetProducts()
{
    var products = await _dbContext.Products
        .Where(x => !x.IsDeleted)
        .ToListAsync();

    return _mapper.Map<List<ProductResponse>>(products);
}
```

**==Notices:==** 
Phân tích code: toàn bộ những products chưa bị xóa được **list lên trên ram** (hiện giờ ta thấy được có 4 fields lên ram)
Sau đó thằng Mapper này sẽ, **duyệt qua các products đang được list** lên rồi lại tiếp tục **tạo ra một thằng ProductResponse** nữa. 

Câu hỏi đặt ra: Lỡ product có tới 1000 fields, nhưng trong khi đó response chỉ cần 2 fields, thì cũng cho 998 fields lúc đầu thì nó bị hao ram quá, nên mình cần select để giảm thiểu thôi.

- Tối ưu hơn là ta sẽ select trước. Sau khi select thì chỉ còn có 2 fields.-

```CSharp
public async Task<List<ProductResponse>> GetProducts()
{
	var product = await _dbContext.Products
		.Where(x => !x.IsDeleted)
		.Select(x => new ProductResponse
		{
			Id = x.Id, 
			Name = x.Name, 
			Price = x.Price, 
			CoverImage = x.CoverImage
		})
		.ToListAsync();
		
	return _mapper.Map<List<ProductResponse>>(products);
}
```