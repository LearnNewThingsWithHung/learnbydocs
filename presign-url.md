```HTML
Client (Web/App)                  Backend Server                  Storage (AWS S3)
       │                                │                                │
       │ 1. "Anh ơi, em muốn up         │                                │
       │    ảnh avatar_1.jpg"          │                                │
       ├───────────────────────────────>│                                │
       │                                │ 2. Dùng Access Key tạo ra      │
       │                                │    một URL có kèm "chữ ký"     │
       │                                │    & thời gian hết hạn (5p)    │
       │                                │                                │
       │ 3. Trả về Presigned URL        │                                │
       │<───────────────────────────────┤                                │
       │                                                                 │
       │ 4. HTTP PUT / POST (Đẩy trực tiếp file ảnh bằng URL vừa nhận)   │
       ├────────────────────────────────────────────────────────────────>│
       │                                                                 │ 5. S3 tự check chữ ký,
       │                                                                 │    nếu chuẩn -> Lưu file
       │ 6. Trả về: "Up xong rồi nhé!"                                   │
       │<────────────────────────────────────────────────────────────────┤
```

#### 1.Khi người dùng muốn upload ảnh lên một cái cloud
- Cách 1: 
	- Người dùng gửi file cho Backend, BE nhận file rồi gửi tiếp lên cloud
	- **Vấn đề:**
		- BE phải xử lí file nặng
		- Tốn CPU, RAM, băng thông
		- Hệ thống lớn sẽ rất tốn tài nguyên
- Cách 2: 
	- FE gọi BE và bảo rằng: "Em muốn upload file"
	- BE tạo một URL đặc biệt
	- BE trả URL đó về FE
	- FE dùng URL đó upload thằng lên cloud
	- -> BE không cần cầm file nữa

##### Pre-signed URL thực chất là gì ?
- Pre-signed URL là một URL được backend tạo ra và ký bằng secret key, cho phép client upload hoặc download file trực tiếp từ storage trong một khoảng thời gian giới hạn mà không cần cấp quyền truy cập trực tiếp vào storage. Nó giúp giảm tải cho backend và tăng tính bảo mật.
- Vd:
```TEXT
https://bucket.s3.amazonaws.com/avatar.jpg
?signature=abcxyz
&expires=3600
```
	- Url: này có thể :
		- Chỉ upload được file
		- Hoặc chỉ download được file
		- Chỉ sống trong 1 giờ 
		- Hết hạn thì vô dụng

##### Tại sao không cho người dùng truy cập thẳng S3?
- Người dùng có thể xóa file, sửa file, đọc toàn bộ bucket của mình -> cực mình nguyên hiểm
- Nên lúc này mình chỉ gửi cho nó cái Pre-signed Url để giới hạn quyền cho nó

##### Tại sao phải ký ?
- Vì đa số file trên cloud được lưu ở chế độ private. Backend dùng Secret Key để tạo chữ ký chứng minh người dùng được phép truy cập file đó -> giúp nó có thể upload ảnh lên dễ dàng
- Đồng thời set timeout để tránh việc link lộ và bị sử dụng vĩnh viễn.

#### nó khác biệt gì so với upload thông thường là BE upload ảnh lên
- Upload truyền thống yêu cầu file đi qua backend trước khi được upload lên storage. Điều này làm backend tốn CPU, RAM và network bandwidth mặc dù backend chỉ đóng vai trò trung gian.
- Với Pre-signed URL, backend chỉ tạo một URL có chữ ký và thời gian hết hạn, sau đó client upload trực tiếp lên storage. Backend không phải xử lý file nên hệ thống scale tốt hơn và giảm đáng kể tài nguyên sử dụng.
- Tuy nhiên nếu cần xử lý file như resize ảnh, scan virus hoặc validate nội dung thì vẫn nên upload qua backend trước khi lưu vào storage.
