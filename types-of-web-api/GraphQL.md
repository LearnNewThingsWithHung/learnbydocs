### GraphQL: được sinh ra để giải quyết vấn đề ==FRONTEND & DATA FETCHING==

## A. GraphQL giải quyết vấn đề gì? (Tại sao nó ra đời?)

Cái đau đầu mà backend engineer gặp phải khi hệ thống lớn lên với REST API là hai căn bệnh kinh điển: **Over-fetching (Thừa dữ liệu)** và **Under-fetching (Thiếu dữ liệu)**.

Anh lấy ví dụ thực tế thế này: Em đang làm backend cho một app giống như Instagram.

- **Bài toán Under-fetching:** Ở trang chủ, mobile app cần hiển thị danh sách bài viết (Post). Mỗi bài viết cần có tên tác giả và 3 comment mới nhất. Nếu dùng REST, endpoint `/api/v1/posts` chỉ trả về thông tin bài viết và `user_id`. Để có tên tác giả, client lại phải gọi tiếp `/api/v1/users/{id}`. Để có comment, lại phải gọi `/api/v1/posts/{id}/comments`. Thế là để hiện một cái màn hình, điện thoại của user phải thét lên server 3-4 request HTTP. Mạng yếu một phát là app quay vòng vòng ngay.
    
- **Bài toán Over-fetching:** Để sửa lỗi trên, em gộp hết vào một endpoint "đại phẫu" là `/api/v1/home-feed` trả về tất tần tật. Nhưng ngặt nỗi, một màn hình khác trên app chỉ cần đúng cái `id` và `title` của bài viết thôi. Lúc này, server vẫn còng lưng lôi cả đống dữ liệu nặng nề (nội dung post, thông tin user, mớ comment...) từ Database lên rồi ném qua mạng cho client. Cực kỳ tốn băng thông và làm chậm hệ thống.
    

Thằng Facebook chính là ông tổ đẻ ra GraphQL vào năm 2012 (và opensource năm 2015) vì họ thấm thía nỗi đau này nhất khi chuyển giao giao diện từ Web sang Mobile. Họ cần một cơ chế mà **Client cần gì thì tự định nghĩa, Server chỉ trả đúng bấy nhiêu, tất cả gói gọn trong một Request duy nhất.**

## B. Cơ chế hoạt động của GraphQL

Bản chất ở đây là GraphQL thay đổi hoàn toàn cách Client và Server giao tiếp với nhau.

Thay vì có hàng chục endpoint như REST (`/users`, `/posts`, `/comments`), GraphQL **chỉ có duy nhất một endpoint** (thường là `/graphql`) và tất cả các request đều dùng phương thức **POST**.

Để lấy dữ liệu, Client sẽ gửi một cái "đơn đặt hàng" gọi là **Query** lên endpoint này. Server nhìn vào đơn đó, chạy qua một tầng gọi là **Resolver** để bốc thuốc đúng những gì được yêu cầu rồi trả về dạng JSON.

Em hãy nhìn cái flow chạy thực tế dưới đây:

```
[ Client (Mobile/Web) ]
       │
       │  1. Gửi POST /graphql kèm Body là Query mong muốn:
       │     "Cho tôi xin post có id=1, lấy đúng trường title và name của author thôi"
       ▼
[ API Gateway / GraphQL Server ]
       │
       │  2. Phân tích cú pháp (Parsing & Validating) Schema.
       │  3. Gọi các hàm "Resolver" tương ứng:
       │     - Resolver_Post(id: 1) -> Chọc vào DB lấy Post
       │     - Resolver_Author(user_id) -> Chọc vào DB lấy User
       ▼
[ Database (SQL/NoSQL) ]
       │
       │  4. Trả dữ liệu thô về cho Server
       ▼
[ GraphQL Server ]
       │
       │  5. Gom lại, lọc đúng các trường client xin, đóng gói thành JSON
       ▼
[ Client (Mobile/Web) ] (Nhận đúng JSON sạch sẽ, không thừa không thiếu)
```

## C. Ưu điểm của GraphQL

- **Client làm chủ dữ liệu:** Frontend cần thêm trường gì hay bớt trường gì thì tự sửa trong Query của họ, Backend không cần phải vào code sửa lại API hay tạo endpoint mới. Quá trình phát triển tính năng (development velocity) nhanh hơn hẳn.
    
- **Một Request cân tất cả:** Như ví dụ Instagram ở trên, Client có thể lồng (nest) các mối quan hệ vào nhau trong một Query. Server xử lý ngầm và trả về một cục JSON duy nhất. Giảm tải số lượng kết nối HTTP cực lớn cho thiết bị mobile.
    
- **Strongly-typed (Kiểu dữ liệu nghiêm ngặt):** GraphQL bắt buộc em phải định nghĩa một cái **Schema** (khuôn mẫu) rõ ràng: trường nào là String, trường nào là Int, có được null hay không. Nó giống như một bản hợp đồng ký kết giữa Frontend và Backend, không có chuyện "ông nói gà bà hiểu vịt".
    

## D. Nhược điểm (Cái giá phải trả - Trade-off kỹ thuật)

Thằng này mạnh ở chỗ linh hoạt cho Client, **nhưng đổi lại, nó là cơn ác mộng của Backend** nếu em không hiểu bản chất. Lúc hệ thống lớn lên, vấn đề bắt đầu xuất hiện:

- **Nỗi đau N+1 Queries (Chết đứng Database):** Đây là lỗi kinh điển nhất của GraphQL. Giả sử Client xin danh sách 10 bài viết, và trong mỗi bài viết họ xin thêm tên `author`. Nếu em viết Resolver ngây thơ, server sẽ chạy 1 câu lệnh SQL để lấy 10 bài viết, sau đó lặp qua từng bài viết và chạy tiếp 10 câu lệnh SQL khác để lấy thông tin của từng author. Tổng cộng là $1 + 10 = 11$ câu lệnh chọc vào DB. Nếu là 1000 bài viết? DB sập nguồn ngay lập tức.
    
- **Phức tạp hóa bài toán Cache:** Với REST, em có thể cache ở tầng Network (dùng CDN, Nginx hoặc trình duyệt) rất dễ dàng vì mỗi URL là một tài nguyên cố định (Ví dụ: Cache kết quả của GET `/api/posts/1`). Nhưng với GraphQL, tất cả đều là POST `/graphql`, URL giống hệt nhau, chỉ khác cái Body. Network Cache chịu chết, em buộc phải implement cache ở tầng Application (trong code Backend) phức tạp hơn nhiều.
    
- **Nguy cơ bị DDOS bằng Query hủy diệt:** Vì Client tự do vẽ Query, một hacker cố tình viết một Query lồng nhau vô tận (Post -> Author -> Posts -> Author...) bắt server tính toán liên tục. Nếu không giới hạn độ sâu (Query Depth Limiting), server sẽ bị cạn kiệt tài nguyên (CPU/RAM).
    

## E. So sánh trực diện: REST vs GraphQL

|**Tiêu chí**|**REST API**|**GraphQL**|
|---|---|---|
|**Endpoint**|Nhiều endpoint (`/users`, `/products`...)|Duy nhất một endpoint (`/graphql`)|
|**HTTP Method**|Đa dạng (GET, POST, PUT, DELETE...)|Gần như 100% dùng POST|
|**Định dạng dữ liệu**|Server quyết định cấu trúc trả về|Client quyết định cấu trúc trả về|
|**Caching**|Cực kỳ dễ (Leverage được HTTP Caching nguyên bản)|Khó, phải xử lý ở tầng Application|
|**Tốc độ làm tính năng**|Chậm hơn vì Backend phải viết lại API khi UI thay đổi|Nhanh, Frontend tự điều chỉnh Query|

## F. Khi nào nên dùng & Hệ thống lớn dùng ra sao?

### Khi nào nên dùng?

- Khi em làm các hệ thống dạng Dashboard, App mạng xã hội, hoặc E-commerce nơi mà **giao diện thay đổi liên tục**, các thiết bị (Web, App iOS, App Android, SmartWatch) yêu cầu các lượng dữ liệu khác nhau cho cùng một thực thể.
    
- Khi em xây dựng tầng **BFF (Backend For Frontend)** hoặc **API Gateway** để tổng hợp dữ liệu từ nhiều dịch vụ microservice khác nhau về một mối cho Client tiện dùng.
    

### Hệ thống lớn dùng ra sao?

Các ông lớn như Shopify, GitHub, Netflix, hay Twitter dùng GraphQL ở quy mô cực khủng. Để giải quyết các nhược điểm nêu trên, họ áp dụng các kỹ thuật nâng cao:

1. **Dùng DataLoader:** Để giải quyết bài toán N+1 Queries, họ dùng thư viện DataLoader (cơ chế Batching và Caching). Thay vì chọc DB liên tục, nó sẽ gom tất cả các `user_id` lại rồi chỉ chạy đúng một câu lệnh `SELECT * FROM users WHERE id IN (1, 2, 3...)`.
    
2. **Persisted Queries:** Thay vì cho Client gửi một chuỗi Query dài loằng ngoằng và nguy hiểm, họ bắt Client đăng ký Query đó trước với Server lúc build code. Server lưu Query đó lại và cấp cho nó một cái mã Hash (Ví dụ: `abcdef123`). Lúc chạy thực tế, Client chỉ cần gửi mã Hash đó lên. Vừa bảo mật, vừa tiết kiệm băng thông.
    

## G. Sai lầm người mới (Junior) hay gặp nhất

1. **Mang tư duy Database mapping thẳng lên GraphQL Schema:** Thiết kế Schema giống hệt cấu trúc bảng trong SQL. Bản chất Schema phải được thiết kế dựa trên **góc nhìn của UI/Client**, Client cần đọc dữ liệu thế nào thì thiết kế Schema thế đó, đừng bê nguyên xi kiến trúc DB lên.
    
2. **Quên không xử lý N+1 Query:** Cứ hí hửng vì code chạy mượt ở môi trường Local (vì Data ít), đến khi đem lên Production, lượng User tăng, DB bắt đầu nghẽn cổ chai (Bottleneck) vì gánh hàng triệu câu lệnh SQL thừa thãi.
    
3. **Dùng GraphQL cho các tác vụ không phù hợp:** Ví dụ như tính năng Upload File dung lượng lớn. Dù GraphQL có hỗ trợ nhưng xử lý Multipart Form Data qua GraphQL rất cồng kềnh và hiệu năng kém. Những case này, các Senior luôn tách riêng ra một REST endpoint truyền thống hoặc sinh Presigned URL từ S3 ném cho Client tự upload thẳng lên Cloud.