#### SOP: Same-Origin Policy
- Sop mà luật mặc định của browser
- Origin: Protocol + Domain + Port
Browser mặc định: 
=> Nếu khác origin thì không cho js đọc dữ liệu từ Origin khác

#### CORS: Cross-Origin Resource Sharing
Hiểu đơn giản: cơ chế sủa Server nói với Browser:
"Tôi đồng ý cho thằng mà khác origin với tôi có được dữ liệu của tôi"

Minh họa:
FE:
```JavaScript
fetch(
  "https://api.mycompany.com/users"
)
```
Browser tự động gửi:
```HTML
GET /users

Origin:
https://app.mycompany.com
```
BE trả:
```HTML
Access-Control-Allow-Origin:
https://app.mycompany.com
```
Browser kiểm tra Origin gửi lên == Origin mà thằng kia gửi request xuống, nếu ok thì cho Js đọc được response gửi về

=> CORS là cơ chế của browser cho phép domain nào có thể đọc response của server. Nó được xây dựng dựa trên Same-Origin Policy. CORS không ngăn request tới backend mà chỉ ngăn JavaScript đọc dữ liệu trả về. Với các request non-simple như PUT, DELETE, hoặc có custom header như Authorization, browser sẽ gửi một preflight OPTIONS request để hỏi server trước khi gửi request thật.
