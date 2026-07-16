Khi làm việc với json, có 2 khái niệm code lỗi, mà bạn cần phải nắm:
- Serialization: Parse Object -> Json | để gửi đi hoặc lưu DB
- Deserialization: Parse Json -> Object | để chấm . lấy thuộc tính
#### Các key word thường gặp khi làm việc với json
1. **Json Root ?**
- Ví dụ: đây là một chuộ json thực tế
```Json
{
  "order_id": "AHA-123",
  "status": "COMPLETED",
  "distance": 5.4,
  "contact": {
    "name": "An",
    "phone": "090"
  }
}
```

Trong CSharp, khi bạn yêu cầu máy tính đọc chuỗi json này bằng lệnh **JsonDocument.Parse(json)**, thì máy tính sẽ dựng lên một mô hình dạng cây trong bộ nhớ để quản lí.

- **Root**(gốc): Chính là toàn bộ dấu ngoặc nhọn { } ở ngoài cùng, nó đại diện như là điểm bắt đầu của cả cái cây dựng lên này.
- **Trong code**, để chạm nào điểm bắt đầu này, bạn dùng thuộc tính **RootElement**(phần tử gốc), Từ gốc này, bạn mới có thể đi tìm các thông nhỏ hơn nằm bên trong root như: order_id, status, distance hay contact.

2. **JsonElement**?
- Khi bạn đang đứng ở Root, thì mọi thứ bạn nhìn thấy và chạm vào bên trong JSON đều được gọi chung là một **JsonElement**.
- JsonElement này, đóng vai trò như mà một **chiếc hộp đa năng**, nó có thể đại diện cho bất kì dữ liệu nào trong thế giới Json, như: string, số, object, array, logic.
- Sau đó, bạn phải tự "mở hộp" để lấy kiểu dữ liệu thật sự ra bằng các lệnh như .GetString(), .GetDouble() hoặc .GetInt32(), ...

Ví dụ, bây giờ muốn lấy giá trị bên trong key: order_id thì làm như thế nào?
```CSharp
using(JsonDocument document = JsonDocument.Parse(json))//chuyển json thành một cái cây trong bộ nhớ
{
	//Đứng ở root(gốc) của cái cây, để bạn có thể truy cập vào thuộc tính khác
	JsonElement root = doc.RootElement;
	//nhảy vào thuộc tính order-id
	JsonElement orderIdElement = root.GetProperty("order_id");
	//lấy giá trị
	var orderIdValie = orderIdElenment.GetString();
	//
	cw(orderIdValie); //"AHA-123"
	
}
```
-Nếu muốn lấy giá trị của Value name?
```CSharp
using(JsonDocument doc = JsonDocument.Parse(json))
{
	JsonElement root = doc.RootElement;
	JsonElement contactElement = root.GetProperty("contact");
	JsonElement nameElement = contactElement.GetProperty("name");
	
	var name = nameElement.GetString();

}
//nếu ko có contact thì sẽ bị crash, nên thay vì dùng GetProPerty -> TryGetProperty

```
- Phiên bản ngắn gọn:
```CSharp 
using(JsonDocument doc = JsonDocument.Parse(json))
{
	var name = doc.RootElement
				.TryGetProperty("contact")
				.TryGetProperty("name")
				.GetString();
}

```

3. **JObject**?: Json động
Thằng này nó tương tự nhon JsonDocument, chúng khác nhau ở chỗ: **JsonDocument** chỉ dùng để lấy dữ liệu còn **JObject** có thêm khả năng chỉnh sửa dữ liệu đó nữa
Mình sửa trực tiếp luôn, mà ko trần parse sang Object
- Ví dụ:
```Json
{
  "OrderId": "AHA-123",
  "DepotId": 999,
  "Status": "PENDING"
}
```
- Muốn sửa DepotId  thành 1, và status: thành Pending
```CSharp
var order = JObject.Parse(json);
order["DepotId"] = 1;
order["Status"] = "SHIPING";
order.Remove("OrderId");
```

#### Thực tế
- Gửi một request sang Ahamove -> parse sang Object + log rỗi ra xem
```CSharp
//bạn đang gửi một cái request lên cho ahamove
var ahamoveHttpRequest = new HttpRequestMessage(HttpMethod.Post, url);
ahamoveHttpRequest.Header.Add("Authorization", ahamoveToken);
ahamoveHttpRequest.Content = JsonContent.Create(request);
//hứng dữ liệu trả về
HttpResponseMessage  ahamoveResponse = await _httpClient.SendAsync(a
hamoveRequest);
//bên trong cái HttpResponseMessage: có messageCode + Content + header
//lấy content ra
//xem data debug
var debug = await ahamoveResponse.Content.ReadAsStringAsync();
Console.WriteLine($"[DEBUG NHANH.VN RAW DATA]: {debug}");
var data = await ahamoveResponse.Content.ReadFromJsonAsync<response>();

```
