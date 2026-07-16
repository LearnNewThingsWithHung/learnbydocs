##### Có code tay, hay đọc hiểu code là được, chuyện tương lai
### Register
- Flow Register
#### Vấn đề 1: hashed-password 
##### Thuật toán băm dữ liệu chung: MD5, SHA-1, SHA-256, SHA-512
Biến đổi một chiều, các hacker:
- **Brute Force**: đoán rồi hash từng cái
- **Dictionary Attack**: phạm vi nhỏ, những password phổ biến rồi, cũng vừa đoán vừa hash
- **Rainbow Table**: hash một lần, lấy được db tra ngược lại lên file đã hash đó
=> sinh ra **salt** để chống **Rainbow Table**:
```HTML
Giải thích chi tiết:
Rainbow Table: chuẩn bị các password, hash một lần, bỏ trong một cái file
Khi hacker nó lấy được cái password hash ở db, nó tra ngược lại trên cái file đó.
Đó là khi chưa có salt.
Bây giờ có salt rồi, thì password dưới db và cái file mà nó đã hash được, là 2 cái hoàn toàn khác nhau.
Nếu nó lấy được cái salt, nó phải lấy hoàng loạt cái password của nó + với cái salt đó ra hoàng loạt cái hashed password, mà chỉ để check một cái hassed password => ko thể, dung lượng quá lớn
```

Vậy dùng MD5, SHA-1, SHA-256, SHA-512 để hash password có đủ an toàn không?
-> Không, vì bản chất MD5 và SHA được sinh ra để kiểm tra file có bị sửa không, dữ liệu trong file có toàn vẹn không. Và SHA-256 rất nhanh -> điều đó làm cho hacker thử password cực nhanh

Tóm gọn lại:
- Hacker gần như không "giải mã ngược" MD5 hay SHA-256.
- Họ **đoán password rồi hash lại để so sánh**.
- MD5 và SHA-1 hiện nay quá yếu.
- SHA-256 mạnh về mặt mật mã nhưng vẫn quá nhanh để lưu password.
- Hệ thống hiện đại thường dùng, để làm chậm:
    - BCrypt
    - Scrypt
    - Argon2 (được xem là lựa chọn rất tốt hiện nay)

**Thế hệ 1: BCrypt**
Dùng phổ biến hiện nay
Ý tưởng: Muốn brute force khó thì làm cho hash chậm đi.

**Thế hệ 2: Scrypt**
Ý tưởng: Không chỉ tốn CPU mà còn phải tốn RAM.

**Thế hệ 3: Argon2**
Ý tưởng: CPU + RAM  + Parallelism đều được kiểm soát

