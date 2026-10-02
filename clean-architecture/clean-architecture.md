
![[Pasted image 20261001061925.png]]
Trong thực tế, chúng ta hay bắt gặp rất nhiều kiến trúc phần mềm, như:
+ N-Tier Architecture (dựa án TetPee)
+ Clean Architecture (là một kiến trúc sạch, lấy domain làm trung tâm)
+ Microservices (chia hệ thống lớn làm các services nhỏ -> liên kết chúng với nhau)
+ CQRS 
+ MVC, ...
=> Trong các dự án thực tế, người ta thường không hay sử dụng đúng (i sì) những kiến trúc phía trên, mà thường biến tấu để phù hợp hơn với những dự án khác nhau. 

Ví dụ: tôi học Clean Architecture -> nhưng khi tôi áp dụng lại dùng Clean Architecture A'

Vậy bạn chất ở đây không phải là ta học mô hình nào, ta sẽ áp dụng y chang mô hình đó
- Không phải lên mạng, search "Clean Architecture builing", rồi thấy: "À", nó phân ra làm 4 tầng nè, tao sẽ dùng API để hiểu 4 tầng đó, rồi sau đó tao dựng src rồi code một phát là được á mà. Quá  "easy". 
- Nhưng khi đụng chạm vào thực tế, điều đó thường khó xảy ra (như mình nói ở trên: không phải lúc nào cũng 4 tầng mà học vẹt), bản chất là bạn phải hiểu được kiến trúc clean là một kiến trúc như thế nào, để bạn có thể triển khai theo ý mình muốn. Nó không nhất thiết phải là 4, mà nó có thể là bất cứ thứ gì -> Vậy điều cần đặt ra câu hỏi ở đây là, làm sao chúng ta biết được, cái kiến trúc mà chúng ta thiết kế là "đúng",  là "clean" 

### I. Theo mình, danh sách các tiêu chí cho **một kiến trúc phần mềm** :
1. **Module hóa các thành phần**
	- Tại sao ? Bởi vì một kiến trúc được xem là tốt khi các thành phần của chúng phải được tách biệt với nhau. Nếu nó dính chùm với nhau thì sau này khó scale và phát triển được
	- Bằng chứng là mô hình Tier-layer mà chúng ta đã học, nó được module hóa thành 3 phần riêng biệt: Controller, Service, Repo
	
2. **Giảm tối thiểu sự phụ thuộc vào các công nghệ (Famework, Database, Library)**
	- Trường hợp hợp này hơi hiếm, tại đa phần chúng ta sẽ chốt công nghệ với nhau rồi mới bắt tay vào làm
	- Nhưng "lỡ", trong quá trình làm thì thấy Postgres nó không đáp ứng được nhu cầu của chúng ta nữa, ta cần chuyển sang MongoDB, thì lúc này quay ra ta thấy layer Repository của chúng ta sẽ bị sâp hết, chưa kể, tầng Service cũng bị ảnh hưởng. Có 3 tầng, mà thay đổi một cái công nghệ -> khả năng cao là dựng lại src code luôn
	=> Mở rộng hơn ý, đó là, giảm thiểu tối đa sự phụ thuộc vào những gì có khả năng bị thay đổi
	
3. **Dễ test, dễ kiểm thử**
	- Cái này là một trong những vấn đề của hệ thống HTM của chúng ta. Nhiều khi chỉ cần test là "sản phẩm hết hàng, tức số lượng là 0", nhưng thường phải trải qua quá trình, mock dữ liệu giả, mock db, add giữ liệu mà db, tạo service, gọi hàm, ném lỗi -> "Ôi", tôi chỉ test đó là số lượng sản phẩm = 0 mà quăng ra lỗi, thì liên quan đến đến việc tạo db, tạo service, chưa kể còn nhiều cái trường hợp ảo ảo, trong service, constructure chỉ cần thêm một DI nữa, là trong unit test của phần đó bị lỗi hết..
	
4. **Thích nghi với sự thay đổi**
	- Trong thực tế phát triển phần mềm, có những khách hàng, người ta thực sự cũng không biết rằng họ đang muốn thứ gì(-> thường xuyên thay đổi ý tưởng), hoặc là mình có thể hiểu sai requirement của họ. 
	- -> Dẫn đến, phải update thay đổi thay nhu cầu của khách hàng, với 1 kiến trúc code khi găp những tình huống như trên mà ko thay đổi được, hoặc thay đổi mất rất nhiều thời gian -> kiến trúc này gãy.
	=> Khá gần với topic dễ mở rộng

### II. Cơ bản Clean Architecture
"Clean" + "Architecture" -> Kiến trúc, sạch | các thành phần không phụ thuộc lẫn nhau

Theo như mình tìm hiểu, thì kiến trúc này sẽ có 3 tư tưởng cốt lỗi, mà chúng ta nên focus vào, tránh trường hợp học lang mang

1. Đặt logic nghiệp vụ (Business logic) làm trung tâm
	- Khi chúng ta làm một cái phần mềm để phục vụ người dùng, mà người dùng họ cần gì ?
	- Họ cần cái logic nghiệp vụ của họ được số hóa lên phần mềm, được chạy trên các thiết bị internet -> thì đặt là trung tâm là quá chuẩn rồi

2. Dễ kiểm thử (như trên)
3. Độc lập với các công nghệ (như trên)

### III. Phân tích các thành phần(bám vào hình ảnh phía trên)
1. **Thành phần cốt lỗi (chứa logic nghiệp vụ): Layer Entities (Enterprise Rules)**
	- Khi triển khai thì thằng này thường chứa các **đối tượng nghiệp vụ (object)**(giống với các đối tượng nằm trong tầng Repo ở folder Entity của chúng ta)
		
	- Ngoài các object, thì nó cũng có thể là các cái hàm, các cái cấu trúc dữ liệu -> để phục vụ cho nghiệp vụ của mình

	-  Các object (class) và các hàm: mình có thể bổ sung các business rule vào
		- `Ví dụ: ta có một đối tượng là đơn hàng đi. Thì trong order nó sẽ chứa một List<Products> . Và bên trong cái order nó cũng có thể chứa một cái hàm business rule: isOrderValid return về count<List<Products>> phải lớn hơn 0 -> true thì valid | false -> văng lỗi.`
	
```CSharp
	public sealed class Customer : Entity
	{
	    private Customer(Guid id, string name, Email email) : base(id)
	    {
	        Name = name;
	        Email = email;
	    }
	
	    public string Name { get; private set; }
	    public Email Email { get; private set; }
	
	    public static Customer Create(string name, Email email)
	    {
	        var customer = new Customer(Guid.NewGuid(), name, email);
	
	        customer.RaiseDomainEvent(new CustomerCreatedDomainEvent(customer.Id));
	
	        return customer;
	    }
	
	    public void UpdateName(string name)
	    {
	        if (string.IsNullOrWhiteSpace(name))
	        {
	            throw new DomainException("Customer name cannot be empty.");
	        }
	
	        Name = name;
	    }
	}

```
	
	- Tuy nhiên, trong thực tế, người ta thường chỉ triển khai dừng lại ở các đối tượng nghiệp vụ thôi, ít thấy đặt hàm hay cấu trúc dữ vào bên trong nó. Mà người ta thường sẽ quy hoạch nó sang package khác thường ở dạng Helper gì đó, để phục vụ cho app 

2. **Layer Use case (Application Business rule)**: là lớp sẽ implement các bước cụ thể (element) trong quy trình của use case đó
	Use case là gì ? -> giống như là một chức năng mà khách hàng/ hệ thống cần. (Khá tương tự user story: là dùng để mô tả tính năng đó)

	- Ví dụ: Mình có một UC về đăng kí nguồi dùng thì nó sẽ có các bước:
	- -> Xử lý user infor input -> Validate -> Hash password -> ... -> Lưu xuống db

3. **Layer Interface Adapte**r (tương ứng với architecture: MVC): Controller, Gateways, Presenter

	- **Controller**: nó có thể là **UI** hoặc **Endpoint**  (là đầu vào mà người dùng tương tác | FE gọi xuống) -> đi vào Controller đầu tiên -> Use Case .
		- Controller sẽ điều hướng xuống -> xuống Use case tương ứng nào
		- Minh họa![[Pasted image 20261001074729.png]]

	- **Presenter**: sau khi Use case làm việc xong -> sẽ ra được kết quả (data)
			-  Và present nó sẽ nhảy vào và lấy data này và nó se public ra ngoài và presenter sẽ design hiển thị theo kiểu gì đó
		- Minh họa![[Pasted image 20261001075501.png|606]]
		
	- **Gateways**: đừng nhầm lẫn với gatewasy trong microservices (đóng vai trò điều hướng các request vào các services tương ứng)
		- Trong architecture, gateways thường làm việc với data -> **Database** hoặc **third party**
		- Những cái code làm việc với database: query, select, .. thì nó được nằm trong tầng này.
		- Khi làm việ với database chúng ta thường nghĩ tới design pattern Repository, còn data lấy từ bên thứ 3 thường gọi là Service. Do đó, khi nói đến repository hoặc service ta sẽ nghĩ ngay đến package gateways. 

		- Khi làm việc với database ta sẽ làm việc với các ==table== và những table này nó phải ==mapping== với những các ==object(model object)==
		- Nhưng cái object(model object) này với cái object(domain object) nằm trong entity\
		- Domain object nó chỉ chứa những cái key liên quan đến nghiệp vụ thôi
		- Còn Model object nó có thể chứa những thông tin về database
		-> ==Có thể đẻ thêm một thằng package nữa là **Model** -> các model object -> dùng để mapping với Db==
 > Đọc tới khúc này cứ tưởng là ngon ơ, nhưng có vấn đề phát sinh nè

>[!warning]
> Ta có một usecase {
> 		Db.FindProductById(productId)
> }

Thì theo như chúng ta học, có phải thằng này **Db.FindProductById(productId)** -> nó sẽ trả về một model đúng không (giống như bên 3-tier của chúng ta nó trả về class response). 

Nhưng nhưng, hãy nhìn vào hình ảnh lúc đầu đi, thằng ==use case==  nó phụ thuộc vào thằng bên trong là **entity (domain)**, nó không thể phụ thuộc vào thằng model object được, nếu vậy thì sẽ bị phá vỡ đi cái ta  đã đề ra ban đầu là giảm sự phụ thuộc vào những thành phần có khả năng dễ thay đổi.

Vì vậy giải pháp là tiếp tục sinh ra một pagekage nữa (**Mapper**) nó giúp tôi chuyển đổi từ: domain object -> model và ngược lại (giống với class response của chúng của ta á: query ra thực thể entity và map ra response class (DTO)).

4. **Framework and Driver(Nơi tập hợp đồ chơi)**
	- Nó là các framework: Reacts, EF core, ...
	- Các libraries
	- Những cái code hepler

Viết đến đây, H mới nhận thây được rằng là, không phải dự án nào nó cũng có structure như thế, tùy vào độ rộng, độ phức tạp mà chúng ta thiết kế thêm hoặc lượt bỏ nếu quá phức tạp, suy cho cùng thì chúng ta phải hiểu được sự tương tác giữa chúng từ đó các layer mới được sinh ra

Định dừng lại rồi, nhưng mà bonus thêm cho ae nào có muốn học và hiểu về nó ở phần dưới

### Dependence rule
Nếu nhìn vào bức ảnh ban đầu, ta sẽ thấy các mỗi tên đi từ ngoài vào trong, tầng bên ngoài thì phụ thuộc vào tầng kề bên trong nó. 
Ông bên trong không biết gì về ông bên ngoài -> tạo ra được tính độc lập giữa các layer

>[!Important]
>(inner layer) <#> (outter layer)

Ví dụ:
```CSharp
class Controller {
	gọi UseCase
}
-> cái này hợp lí, đi từ ngoài gọi vào trong 
-------
class UseCase
{
	gọi Controller
}
-> ỉa chỉa ngay | nhưng vẫn có cách -> dùng Denpendence Inversion -> đảo ngược

class UseCase
{
	gọi IController
}
-> đảo ngược sự phụ thuộc, usecase này nó gọi thằng interface IController là thằng implement cái interface đó là: Conller -> tạo ra 

UseCase
   |
   v
IController
   ^
   |
Controller 

```


