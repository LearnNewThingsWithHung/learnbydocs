## A. WebSocket giải quyết vấn đề gì? (Tại sao nó ra đời?)

Cái đau đầu mà backend gặp phải khi hệ thống yêu cầu **dữ liệu cập nhật theo thời gian thực (Real-time)** chính là **bản chất "bị động" và "bất đối xứng" của giao thức HTTP/1.1**.

Em cứ tưởng tượng giao thức HTTP/1.1 truyền thống giống như việc em đi ăn nhà hàng nhưng ông đầu bếp bị điếc.

- Client (Em) phải chủ động đập bàn hỏi: _"Có món chưa?"_ -> Server (Đầu bếp) trả lời: _"Chưa có"_.
    
- 5 giây sau em lại đập bàn hỏi: _"Có món chưa?"_ -> Server: _"Chưa"_.
    
- 5 giây sau nữa em lại hỏi -> Server: _"Có rồi đây, cầm lấy"_.
    

Trong mạng máy tính, kỹ thuật "đập bàn hỏi liên tục" này gọi là **Short Polling** hoặc **Long Polling**. Cứ vài giây, Client lại tạo một request HTTP gửi lên server chỉ để hỏi _"Có dữ liệu mới không?"_.

> **Nỗi đau hệ thống ở đây là gì?**
> 
> 1. **Lãng phí tài nguyên kinh khủng (Overhead):** Mỗi cái request HTTP gửi đi không chỉ có mỗi cái câu hỏi trống không, nó gánh theo một cái **HTTP Header** rất nặng (chứa Cookie, User-Agent, Token Auth...). Server phải tốn tài nguyên mở kết nối, đọc Header, check DB, thấy không có gì mới lại đóng kết nối. Hệ thống có 10.000 user cùng chọc liên tục như vậy là server cạn kiệt tài nguyên (CPU/RAM) ngay dù chẳng có dữ liệu nào thực sự được truyền đi.
>     
> 2. **Độ trễ cao (Latency):** Nếu em set 5 giây hỏi một lần, thì dữ liệu có ở giây thứ 1, user vẫn phải đợi đến giây thứ 5 mới nhận được. Với app chat hay app trade coin, chậm 4 giây là user xóa app ngay.
>     

Hệ thống cần một cơ chế mà **Server phải chủ động đẩy dữ liệu xuống cho Client ngay khi có biến động, không đợi Client hỏi.** Đó là lý do **WebSocket** ra đời vào năm 2011.

## B. Cơ chế hoạt động của WebSocket

Bản chất của WebSocket là nó thiết lập một đường ống dẫn dầu (Connection) **hai chiều (Full-Duplex)** và **bền vững (Persistent)** giữa Client và Server. Một khi ống đã thông, hai bên thích bắn dữ liệu cho nhau lúc nào cũng được, không cần thủ tục xin phép rườm rà nữa.

Flow hoạt động của một kết nối WebSocket từ lúc bắt đầu đến lúc truyền dữ liệu sẽ chạy qua các bước sau:

Plaintext

```
[ Client (Browser/App) ]                                   [ Server (Backend) ]
          │                                                         │
          │ 1. Bắn request HTTP truyền thống                        │
          │    "Anh ơi, nâng cấp lên WebSocket đi"                  │
          │    (HTTP GET /chat kèm Header Upgrade)                  │
          ▼                                                         ▼
[ Tầng Xử Lý HTTP ]                                        [ Tầng Xử Lý HTTP ]
          │                                                         │
          │ 2. Server đồng ý, trả về HTTP 101                       │
          │    "Ok em, thông ống luôn!"                             │
          ▼                                                         ▼
    ================ BẮT ĐẦU QUÁ TRÌNH HANDSHAKE (BẮT TAY) THÀNH CÔNG ================
    (Giao thức HTTP chính thức bị khai tử trên đường truyền này. 
     TCP Socket được giữ mở hoàn toàn, chuyển sang giao thức ws:// hoặc wss://)
          │                                                         │
          │ 3. User A nhắn tin: "Hello"                            │
          ├───────────────── (Dữ liệu dạng Frame siêu nhẹ) ────────>│
          │                                                         │ (Server xử lý, thấy User B 
          │                                                         │  đang online trên một ống khác)
          │ 4. Server chủ động đẩy tin nhắn xuống cho User B        │
          `<──────────────── (Không cần User B hỏi trước) ──────────┤
```

### Chi tiết luồng chạy (Under the hood):

1. **Handshake (Bắt tay):** Đầu tiên, Client vẫn gửi một request HTTP GET bình thường lên server. Nhưng trong Header của request này có 2 trường đặc biệt: `Upgrade: websocket` và `Connection: Upgrade`. Nó giống như lời đề nghị: _"Em muốn đổi cách nói chuyện"_.
    
2. **Protocol Switch (Chuyển đổi giao thức):** Nếu server hỗ trợ, nó sẽ trả về mã HTTP Code **101 Switching Protocols**. Ngay tại khoảnh khắc này, cái giao thức HTTP ban đầu biến mất. Kết nối TCP bên dưới được giữ nguyên (không bị ngắt). Kể từ đây, hai bên nói chuyện bằng giao thức `ws://` (hoặc `wss://` nếu có mã hóa SSL/TLS).
    
3. **Truyền nhận dữ liệu (Data Transfer):** Dữ liệu lúc này không bọc trong gói HTTP nặng nề nữa. Nó được chia thành các gói cực nhỏ gọi là **Data Frames**. Một frame WebSocket chỉ tốn khoảng **2 đến 10 bytes** cho phần Header (so với hàng KiloBytes của HTTP Header). Nhờ vậy, tốc độ truyền tin nhanh như chớp.
    

## C. Ưu điểm của WebSocket

- **Real-time thực thụ, độ trễ cực thấp:** Server có data là bắn xuống ngay, Client có data là bắn lên ngay. Độ trễ gần như bằng 0 (chỉ phụ thuộc vào tốc độ mạng vật lý).
    
- **Tiết kiệm băng thông tối đa:** Nhờ lược bỏ được mớ HTTP Header cồng kềnh ở mỗi request, lượng băng thông tiêu thụ giảm đi hàng chục lần khi hệ thống chạy ở quy mô lớn.
    
- **Kết nối 2 chiều bình đẳng:** Server không còn là thằng "gọi mới thưa" nữa. Server giờ đây có quyền lực chủ động tương đương Client.
    

## D. Nhược điểm (Trade-off kỹ thuật và nỗi đau hệ thống)

Thằng này cực sướng cho trải nghiệm người dùng, nhưng là **bài toán quản lý bộ nhớ đau đầu cho Backend**:

- **Bài toán Stateful (Tốn RAM kinh hoàng):** Với REST API (Stateless), Server nhận request xong, xử lý, trả về rồi quên luôn user đó là ai, RAM được giải phóng. Nhưng với WebSocket (Stateful), Server phải giữ mở kết nối cho **tất cả** user đang online. Nếu hệ thống có 100.000 user online cùng lúc, Server phải duy trì 100.000 cái TCP connection trong bộ nhớ. Mỗi kết nối ngốn một lượng RAM nhất định $\rightarrow$ Server rất dễ bị cạn kiệt RAM (Out of Memory).
    
- **Khó Scaling (Mở rộng hệ thống):** Khi một server không gánh nổi lượng connection, em phải nâng cấp lên nhiều server (Horizontal Scaling) đứng sau một Load Balancer. Lúc này phát sinh vấn đề: User A kết nối vào Server 1, User B kết nối vào Server 2. Khi User A gửi tin nhắn cho User B, Server 1 tìm trong bộ nhớ không thấy User B đâu cả. Hệ thống bị "mất liên lạc". Em buộc phải dùng thêm một tầng Message Broker (như Redis Pub/Sub) để làm cầu nối giữa các Server với nhau.
    
- **Không thân thiện với Firewall và Proxy:** Một số hệ thống mạng doanh nghiệp nghiêm ngặt hoặc các con Proxy cũ không hiểu giao thức HTTP 101 Upgrade, chúng nó sẽ tự động ngắt kết nối WebSocket vì tưởng đây là một cuộc tấn công mạng hoặc kết nối lậu.
    

## E. So sánh trực diện: REST vs GraphQL vs gRPC vs WebSocket

Để em phân biệt rõ các loại WebAPI này dưới góc nhìn kiến trúc, hãy xem bảng đối chiếu này:

|**Tiêu chí**|**REST API / GraphQL**|**gRPC**|**WebSocket**|
|---|---|---|---|
|**Mô hình giao tiếp**|Request - Response (1 chiều)|Request - Response hoặc Streaming|**Bi-directional (2 chiều liên tục)**|
|**Trạng thái kết nối**|**Stateless** (Cắt đuôi ngay sau khi trả dữ liệu)|Gắn kết trên HTTP/2|**Stateful** (Giữ kết nối vĩnh viễn cho đến khi ngắt)|
|**Bản chất đường truyền**|Client hỏi - Server trả lời|Gọi hàm từ xa / Đẩy data luồng|**Hai bên bắn dữ liệu cho nhau bất đối xứng**|
|**Header Overhead**|Rất nặng (Hàng KB cho mỗi request)|Rất nhẹ (HTTP/2 HPACK compression)|**Siêu nhẹ (Chỉ vài Bytes cho mỗi khung dữ liệu)**|
|**Use case cốt lõi**|Lấy dữ liệu tĩnh, làm tính năng CRUD thông thường|Giao tiếp nội bộ siêu tốc giữa các Microservices|**Các tính năng cần Real-time, cập nhật liên tục từng mili-giây**|

## F. Khi nào nên dùng WebSocket?

Em chỉ nên lôi vũ khí hạng nặng WebSocket ra xài khi hệ thống có các tính năng bắt buộc phải Real-time:

- **Ứng dụng Chat/Message:** Messenger, Slack, Zalo (Tin nhắn đến phải hiện ngay lập tức).
    
- **Bảng điện tử tài chính / Crypto / Stock:** Giá nến nhảy liên tục theo từng mili-giây.
    
- **Game nhiều người chơi (Multiplayer Game):** Tọa độ của nhân vật phải được đồng bộ liên tục giữa các máy chơi game.
    
- **Tính năng cộng tác thời gian thực (Collaborative tools):** Google Docs, Figma (Nhiều người cùng gõ chữ trên một màn hình).
    

## G. Hệ thống lớn dùng ra sao?

Tại các hệ thống lớn, người ta không bao giờ dùng một server duy nhất để vừa gánh REST API vừa gánh WebSocket. Họ sẽ tách riêng **WebSocket Server** ra thành một cụm độc lập (gọi là Real-time Cluster).

Anh lấy ví dụ kiến trúc hệ thống Chat của một app lớn:

Plaintext

```
[ Client A (Kết nối vào) ]                [ Client B (Kết nối vào) ]
           │                                          │
    (ws:// Server 1)                           (ws:// Server 2)
           ▼                                          ▼
   [ WebSocket Server 1 ]                     [ WebSocket Server 2 ]
           │                                          │
           └───────────► [ Redis Pub/Sub ] ◄──────────┘
                         (Kênh chung "Chat-Room-1")
```

1. **Tách biệt hạ tầng:** Cụm `WebSocket Server` được cấu hình tối ưu hóa cho việc giữ kết nối (Nâng giới hạn file descriptor của OS, tối ưu hóa RAM).
    
2. **Dùng Redis Pub/Sub làm chất keo gắn kết:** Khi Client A gửi tin nhắn cho Client B, `WebSocket Server 1` nhận được tin. Nó không tự tìm Client B, nó ném tin nhắn đó vào một kênh (Channel) trên **Redis**. `WebSocket Server 2` đang "đăng ký" (Subscribe) kênh đó, nó nhặt được tin nhắn và thấy _"À, Client B đang kết nối vào tao đây rồi"_, nó chủ động đẩy xuống cho Client B. Nhờ kiến trúc này, hệ thống có thể mở rộng ra hàng trăm WebSocket Server mà không sợ mất kết nối giữa các user.
    

## H. Sai lầm người mới (Junior) hay gặp nhất

1. **"Thần thánh hóa" WebSocket, dùng cho mọi thứ:** Thấy WebSocket real-time sướng quá, làm trang tin tức hiển thị bài viết cũng dùng WebSocket, làm trang đăng ký thành viên cũng dùng WebSocket. Đây là tự sát. Hệ thống sẽ sập vì cạn kiệt RAM do phải nuôi hàng triệu kết nối vô ích, trong khi bản chất tính năng đó chỉ cần REST API gọi 1 lần là xong.
    
2. **Quên không bắt mạch kết nối (Heartbeat / Ping-Pong):** Trong thực tế, kết nối mạng giữa Client và Server có thể bị đứt ngầm (Ví dụ: Đi vào đường hầm, mất sóng 3G). Nếu em không code cơ chế **Ping-Pong** (Server định kỳ gửi gói tin "Mày còn sống không?", Client trả lời "Còn"), Server sẽ tưởng kết nối đó vẫn sống và tiếp tục giữ RAM cho nó. Hậu quả là hệ thống bị rác dữ liệu, tràn bộ nhớ do giữ các **kết nối ma (Zombie Connections)**.
    
3. **Không xử lý Authentication cẩn thận:** Nhiều bạn nghĩ lúc Handshake check auth một lần là xong. Nhưng WebSocket duy trì kết nối rất lâu (có thể cả ngày). Nếu trong ngày đó, Token của User bị hết hạn, hoặc tài khoản bị khóa, Backend nếu không chủ động ngắt kết nối WebSocket của user đó từ server (Force Close) thì họ vẫn tiếp tục nhận và gửi được dữ liệu lậu qua đường ống cũ.
    

Tóm lại: **REST/GraphQL là kiểu "gọi điện thoại" (Nói xong cúp máy). gRPC là "đàm thoại nội bộ phòng ban". Còn WebSocket là "mở một cái bộ đàm cầm tay liên tục" между Client và Server.**