```HTMl
LocalStorage / HttpOnly Cookie
            ↓
		  CORS
            ↓
		Rate Limit
```

Sau khi tìm hiểu về cơ chế **CORS** của browser thì thấy hệ thống có vẻ an toàn rồi.
**Nhưng** CORS nó chỉ chặn ko chi JS đọc repsonse thôi. Chớ nó ko chặn JS gửi request

Vậy nếu một thằng nó  gọi 100.000 request/phút thì sao?
-> Đó chính là lúc Rate Limit xuất hiện

**Rate Limit** là cơ chế **giới hạn số lượng request** mà một client được phép gửi trong một khoảng **thời gian nhất định**. Mục đích là chống spam, brute force, lạm dụng API và bảo vệ tài nguyên hệ thống. Khác với CORS chỉ kiểm soát việc browser có được đọc response hay không, Rate Limit kiểm soát tần suất gọi API của client. Khi vượt quá giới hạn, server thường trả về HTTP **429 Too Many Requests.**


**CORS và Rate Limit** giải quyết hai vấn đề hoàn toàn khác nhau nên vẫn cần dùng cùng nhau. **CORS** **kiểm soát website nào được browser cho phép đọc response từ API**, còn **Rate Limit kiểm soát một client được phép gửi bao nhiêu request trong một khoảng thời gian**. Nếu chỉ có CORS thì một frontend hợp lệ hoặc một script dùng Postman/curl vẫn có thể spam API hàng nghìn lần. Ngược lại, nếu chỉ có Rate Limit thì website không được phép vẫn có thể cố gắng truy cập dữ liệu. Vì vậy CORS và Rate Limit là hai lớp bảo vệ độc lập, không thay thế được cho nhau.

##### 1.Fixed Window (Cửa sổ cố định)
"Mỗi phút phát cho em 100 vé, hết phút phát lại 100 vé mới."
**Ưu điểm:** Dễ triển khai, ít tốn tài nguyên.  
**Nhược điểm:** Có thể bị spam ở thời điểm chuyển cửa sổ (cuối phút và đầu phút).

##### 2. Sliding Window (Cửa sổ trượt)
"Bảo vệ luôn nhớ 60 giây gần nhất em đã đi qua bao nhiêu lần."
**Ưu điểm:** Chính xác hơn Fixed Window.  
**Nhược điểm:** Tốn bộ nhớ và tài nguyên hơn.

##### 3. Token Bucket (Thùng Token)
Người dùng có một "ví token". Mỗi request tiêu tốn 1 token. Token được nạp lại đều đặn theo thời gian.
**Ưu điểm:** Cho phép burst (bùng nổ request ngắn hạn) nhưng vẫn kiểm soát được tốc độ trung bình.  
**Nhược điểm:** Triển khai phức tạp hơn Fixed Window.

##### 4. Leaky Bucket (Thùng Rò Rỉ)
Request được đưa vào hàng chờ và hệ thống xử lý với tốc độ cố định.
Dù 100 request tới cùng lúc thì hệ thống vẫn xử lý từ từ theo tốc độ quy định.
**Ưu điểm:** Lưu lượng vào hệ thống rất ổn định.  
**Nhược điểm:** Request có thể phải chờ, tăng độ trễ.

#####
**Fixed Window** = reset theo từng khoảng thời gian cố định.  
**Sliding Window** = tính trên khoảng thời gian thực gần nhất.  
**Token Bucket** = cho phép burst nhờ token tích lũy.  
**Leaky Bucket** = ép request đi vào hệ thống với tốc độ đều đặn.