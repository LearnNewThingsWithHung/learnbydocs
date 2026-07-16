## A. gRPC giải quyết vấn đề gì? (Tại sao nó ra đời?)

Cái đau đầu mà backend gặp phải khi hệ thống lớn lên và chuyển dịch từ kiến trúc **Monolith (Đơn khối - tất cả code chung một source)** sang **Microservices (Hệ thống phân tán - chia nhỏ thành hàng chục, hàng trăm dịch vụ chạy độc lập)** chính là: **Tốc độ giao tiếp nội bộ (Internal Communication)** và **Sự đồng bộ về ngôn ngữ**.

Em cứ tưởng tượng thế này: Ở hệ thống Microservices của một app gọi xe giống như Grab:

- Có Service A lo việc tính tiền (viết bằng Golang).
    
- Có Service B lo việc tìm tài xế (viết bằng Python).
    
- Có Service C lo việc quản lý ví tiền (viết bằng Java).
    

Khi có một cuốc xe mới, các Service này phải liên tục gọi nhau để trao đổi dữ liệu. Nếu em dùng **REST API** truyền thống cho tụi nó gọi nhau, hệ thống sẽ gặp 2 "nỗi đau" cực lớn:

1. **JSON quá nặng và chậm:** REST dùng JSON để truyền dữ liệu. JSON là dạng văn bản (Plain Text), con người đọc rất dễ nhưng máy tính thì ghét. Máy tính phải tốn CPU để biến dữ liệu của nó thành chuỗi JSON (Serialization), gửi qua mạng, rồi máy bên kia lại tốn CPU để dịch chuỗi JSON đó ngược lại thành dữ liệu của nó (Deserialization). Khi hệ thống có hàng triệu request mỗi giây, việc dịch qua dịch lại này làm hệ thống cực kỳ chậm và tốn tài nguyên.
    
2. **Mất thời gian viết Code map dữ liệu:** Vì Service A viết bằng Go, Service B viết bằng Python, nên ông làm Service B phải ngồi đọc tài liệu REST của ông Service A, rồi tự viết code cấu trúc dữ liệu bên Python sao cho khớp với JSON của ông Go trả về. Sai một chữ cái là lỗi ngay.
    

Google nhìn thấy nỗi đau này trong chính hệ thống khổng lồ của họ và họ nghĩ: _"Tại sao gọi một Service ở máy tính khác qua mạng lại không dễ dàng và nhanh như việc gọi một cái hàm (function) có sẵn trong code?"_. Đó là lý do họ đẻ ra **gRPC** (g viết tắt cho nhiều từ tùy phiên bản, nhưng bản chất nó là một framework triển khai **RPC - Remote Procedure Call**, tức là gọi một hàm ở một máy tính từ xa).

## B. Cơ chế hoạt động của gRPC

Bản chất ở đây là gRPC dẹp bỏ hoàn toàn khái niệm URL, Endpoint (`/users`, `/posts`) và dẹp luôn cả JSON. Nó hoạt động dựa trên 2 trụ cột cốt lõi: **Protocol Buffers (Protobuf)** và **HTTP/2**.

Anh em mình cùng đi qua Flow hoạt động của một request gRPC từ Service A sang Service B:

Plaintext

```
[ Service A (Golang) ]                                [ Service B (Java) ]
  │                                                     │
  │ 1. Gọi hàm như code local:                          │
  │    CheckWallet(user_id)                             │
  ▼                                                     ▼
[ Generated Stub (Code sinh tự động) ]               [ Generated Skeleton ]
  │                                                     │
  │ 2. Nén dữ liệu thành Binary (Nhị phân)               │ 5. Giải nén Binary 
  │    bằng Protocol Buffers (Cực nhỏ, cực nhanh)       │    thành đối tượng Java
  ▼                                                     ▲
[ Tầng HTTP/2 Transport ]                             [ Tầng HTTP/2 Transport ]
  │                                                     │
  └─────────── 3. Bắn qua đường truyền ────────────────┘
               - Dữ liệu dạng Binary (0101...)
               - Giữ một kết nối duy nhất (Multiplexing)
               - Nén cả Header của gói tin
```

### Flow chạy chi tiết:

1. **Bước 1: Định nghĩa giao kèo (File `.proto`):** Trước khi viết code, kỹ sư của cả 2 Service ngồi lại viết một file gọi là `wallet.proto`. Trong này định nghĩa rõ: Hàm này tên gì, nhận vào tham số gì (ví dụ: `user_id` kiểu `int64`), trả về cái gì (ví dụ: `balance` kiểu `double`).
    
2. **Bước 2: Sinh code tự động (Code Generation):** Dùng công cụ của gRPC để quét file `.proto` này. Nó sẽ tự động sinh ra các đoạn code (gọi là Stub) cho cả ngôn ngữ Go và Java. Kỹ sư không cần tự gõ code map dữ liệu nữa.
    
3. **Bước 3: Thực thi:** Khi Service A gọi hàm, gRPC lấy dữ liệu, nén nó lại thành **dạng nhị phân (Binary)** bằng Protobuf. Dữ liệu nhị phân này siêu nhỏ vì nó chỉ là các dòng bit `0101...` chứ không có các ký tự thừa thãi như ngoặc nhọn `{}`, ngoặc kép `""` của JSON.
    
4. **Bước 4: Truyền tải qua HTTP/2:** Gói tin nhị phân siêu nhỏ này được đẩy qua giao thức **HTTP/2**. Khác với HTTP/1.1 của REST (mỗi request mở một kết nối rồi đóng lại), HTTP/2 mở một kết nối duy nhất giữa 2 Service và giữ nguyên đó để bắn hàng ngàn request liên tục (gọi là Multiplexing), giúp tiết kiệm thời gian khởi tạo kết nối.
    

## C. Ưu điểm của gRPC

- **Hiệu năng (Performance) kinh hoàng:** Nhờ nén dữ liệu dạng Binary và chạy trên HTTP/2, gRPC nhanh hơn REST API từ 5 đến 10 lần, băng thông tiêu thụ giảm đi đáng kể.
    
- **Đa ngôn ngữ nhưng chung một tiếng nói:** Nhờ file `.proto`, team viết Go, team viết Java, team viết Python đều dùng chung một chuẩn. Code gọi nhau mượt mà như thể đang chạy trong cùng một project.
    
- **Hỗ trợ Streaming thực thời cực mạnh:** HTTP/2 cho phép gRPC hỗ trợ 4 kiểu truyền dữ liệu:
    
    - _Unary:_ Một request - một response (giống REST).
        
    - _Server Streaming:_ Client gửi 1 request, Server trả về một dòng dữ liệu liên tục (giống như xem livestream).
        
    - _Client Streaming:_ Client gửi một dòng dữ liệu liên tục, Server xử lý xong trả về 1 response.
        
    - _Bidirectional Streaming:_ Cả 2 bên cùng bắn dữ liệu cho nhau liên tục theo thời gian thực.
        

## D. Nhược điểm (Trade-off kỹ thuật)

Thằng này cực mạnh cho giao tiếp nội bộ, **nhưng đổi lại, nó không phải là viên đạn bạc cho mọi bài toán**:

- **Trình duyệt (Browser) không chơi trực tiếp được:** Các trình duyệt Web hiện tại (Chrome, Safari, Firefox) không có khả năng can thiệp sâu vào HTTP/2 để đọc hiểu các gói tin gRPC nhị phân một cách native. Do đó, em không thể dùng gRPC trực tiếp từ Website của người dùng gọi về Backend được (trừ khi dùng thêm một tầng trung chuyển gọi là gRPC-Web, nhưng rất phức tạp).
    
- **Con người không đọc được dữ liệu:** Với REST hay GraphQL, em dùng Postman gọi thử API là thấy ngay cục JSON chữ nghĩa rõ ràng. Với gRPC, dữ liệu gửi đi là Binary (`0101...`). Nếu không có file `.proto` gốc để dịch ngược lại, em nhìn vào sẽ chỉ thấy một đống ký tự rác rưởi, không thể debug bằng mắt thường được.
    
- **Độ dốc học tập cao (Learning Curve):** Đội ngũ phát triển phải học cách viết file Protobuf, hiểu về HTTP/2, cách cấu hình load balancer cho HTTP/2 (vì load balancer cho HTTP/2 khó cấu hình hơn HTTP/1.1 rất nhiều).
    

## E. So sánh trực diện: REST vs GraphQL vs gRPC

Để em không bị confuse giữa 3 thằng Web API phổ biến này, hãy nhìn vào bảng so sánh bản chất dưới đây:

|**Tiêu chí**|**REST API**|**GraphQL**|**gRPC**|
|---|---|---|---|
|**Giao thức truyền tải**|HTTP/1.1 (chủ yếu)|HTTP/1.1 (chủ yếu)|**HTTP/2 (Bắt buộc)**|
|**Định dạng dữ liệu**|JSON / XML (Text)|JSON (Text)|**Protocol Buffers (Binary)**|
|**Tư duy thiết kế**|Hướng tài nguyên (Resource-oriented)|Hướng đồ thị / Client cần gì xin nấy|**Hướng hành động / Gọi hàm (RPC-oriented)**|
|**Đối tượng sử dụng chính**|Giữa Trình duyệt/Mobile và Backend|Giữa Trình duyệt/Mobile và Backend (Dashboard, App mạng xã hội)|**Giữa các Service nội bộ với nhau (Internal Microservices)**|
|**Độ dễ Debug**|Rất dễ (Nhìn bằng mắt được)|Rất dễ (Nhìn bằng mắt được)|**Khó (Phải có công cụ và file .proto)**|

## F. Khi nào nên dùng gRPC?

- **Giao tiếp giữa các Microservices (Service-to-Service):** Đây là đất diễn hoàn hảo nhất của gRPC. Khi các dịch vụ backend cần nói chuyện với nhau với độ trễ thấp nhất có thể.
    
- **Hệ thống IoT hoặc App Mobile cần tiết kiệm pin/băng thông:** Dữ liệu nhị phân siêu nhỏ giúp các thiết bị phần cứng yếu hoặc mạng chập chờn truyền tải dữ liệu rất nhẹ nhàng.
    
- **Các ứng dụng cần Streaming thời gian thực:** Hệ thống chat, cập nhật vị trí shipper liên tục, hệ thống giám sát (monitoring) bắn log về server trung tâm.
    

## G. Hệ thống lớn dùng ra sao?

Tại các hệ thống microservices quy mô lớn như Netflix hay Uber, họ phân cấp kiến trúc API rất rõ ràng:

Plaintext

```
[ Người dùng cuối: Web Browser / Mobile App ]
                     │
                     │  (Giao tiếp qua REST API hoặc GraphQL - dữ liệu JSON dễ dùng)
                     ▼
             [ API Gateway ]
                     │
                     ├───────────────────────┬───────────────────────┐
                     │ (Dịch từ REST/GraphQL)│                       │ (Dịch từ REST/GraphQL)
                     ▼                       ▼                       ▼
            [ Service Tính Tiền ] ──gRPC──> [ Service Tài Xế ] ──gRPC──> [ Service Bản Đồ ]
                                 (Hàng trăm Microservices nói chuyện nội bộ bằng gRPC)
```

Tức là, **API Gateway** đóng vai trò như một người thông dịch. Nó đứng ở cửa ngõ, nhận request REST/GraphQL (JSON) từ người dùng, sau đó chuyển đổi dữ liệu đó thành các request **gRPC (Binary)** siêu tốc để bắn vào bên trong mạng lưới các Microservices nội bộ. Nhờ vậy, hệ thống vừa thân thiện với Client bên ngoài, vừa đạt hiệu năng tối đa ở bên trong.

## H. Sai lầm người mới hay gặp nhất

1. **Cố đấm ăn xôi dùng gRPC cho Front-end (Web):** Thấy gRPC nhanh quá liền tìm cách bắt ứng dụng ReactJS hay VueJS gọi thẳng gRPC về Backend. Kết quả là phải cài thêm hàng tá thư viện proxy trung gian, khiến code Front-end nặng nề và cực kỳ khó bảo trì.
    
2. **Sửa file `.proto` vô tội vạ làm sập hệ thống (Backward Compatibility):** Trong gRPC, các trường dữ liệu được định nghĩa kèm theo một con số (vị trí - tag number, ví dụ: `string user_name = 1;`). Người mới không hiểu, vào đổi số `1` thành số `2` hoặc xóa bớt trường cũ đi. Khi deploy, Service cũ và Service mới không hiểu nhau nữa, dẫn đến lỗi crash hệ thống hàng loạt. Quy tắc của gRPC là: _Chỉ được thêm trường mới, không được tùy tiện đổi số tag của trường cũ_.