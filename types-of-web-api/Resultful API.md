Rest: là một kiểu kiến trúc, giúp cho các API clean hơn
REST: có 2 ý chính cực lớn:
- URL: đại diện cho tài nguyên (rescource)
```CSharp
/users
/products
/bookings
```
- HTTP method đại diện cho hành động
Vì nhờ nó tách ra làm 2 phần riêng biệt, nên nó trở nên phổ biến hơn
### Rest giải quyết vấn đề thực tế nào?
- Đó là vấn đề ==SERVER & INTERNET SCALE==
	Ngày xưa, khi server muốn biết một request gửi lên là của một user nào thì có 2 cách:
		Stateful: server phải nhớ thằng user đó
		Stateless: user phải tự mang thông tin của nó
	[[../Authentication&Authorization/Stateful & Stateless]] - tìm hiểu thêm
