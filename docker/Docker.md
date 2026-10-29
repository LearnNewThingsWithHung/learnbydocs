# Tài liệu Docker

## 1. Docker là gì?

Docker là công cụ giúp ta có thể chạy project ở một môi trường cụ thể, được định sẵn rõ ràng, độc lập với môi trường gốc. Các ứng dụng chạy trong Docker được gọi là các Container ("Cờn tên nơ")

Trước khi có Docker, chúng ta thường xuyên đối mặt với 2 rắc rối kinh điển:

- **Trường phái "Dự án mới":** Tham gia dự án, phải mất hàng tá giờ đồng hồ để cài đặt môi trường (PHP, MySQL, Redis, Nodejs...). Document cài đặt sơ sài, môi trường local (Windows/Mac) lại khác xa server (Linux) dẫn đến hàng loạt lỗi `BUILD FAILED` không rõ nguyên nhân.
- **Trường phái "Maintain nhiều dự án":** Một máy tính phải chạy 3-4 project, mỗi project yêu cầu một phiên bản database hay thư viện khác nhau. Các tài nguyên dùng chung bị xung đột, gỡ cài đặt một công cụ có thể kéo theo sự sụp đổ của toàn bộ các project khác.

### 2. Docker là gì?

Về mặt định nghĩa kỹ thuật, Docker là nền tảng mã nguồn mở giúp tự động triển khai ứng dụng vào các môi trường ảo hóa.

**Hiểu một cách đơn giản:** Docker là công cụ tạo ra một **môi trường độc lập, được định nghĩa sẵn và bất biến** để chạy project của bạn. Những ứng dụng chạy bên trong Docker được gọi là các **Container**.

**Hiểu theo hệ "thực chiến" (Đơn giản hơn nữa):** Docker giúp chúng ta triển khai các ứng dụng một cách cực kỳ nhanh chóng. Bạn sẽ không còn tốn hàng giờ đồng hồ để setup máy móc, không cần đau đầu suy nghĩ về việc cài đặt môi trường hay xung đột phiên bản. Chỉ cần gõ một dòng lệnh kiểu `docker run "gì gì đó"`, Docker sẽ tự động kéo môi trường về, cài đặt mọi thứ và ứng dụng của bạn cứ thế mà chạy!

### 3. Những "phép màu" Docker mang lại

- **Môi trường "Bất biến" & Tiêu chuẩn hóa:** Định nghĩa môi trường 1 lần (qua file cấu hình), chạy ở mọi nơi (Windows, Mac, Linux) với kết quả giống hệt nhau.
- **Cô lập hoàn toàn (Isolation):** Các project không bị xung đột, không chia sẻ tài nguyên phần mềm lằng nhằng. Bạn có thể cài đặt, sửa xóa thoải mái ở project A mà không ảnh hưởng tới project B. (Giải quyết lỗi Maintain nhiều dự án)
- **Chuyển giao dễ dàng:** Giao project cho người mới chỉ cần ném cho họ file cấu hình Docker, thay vì một file hướng dẫn setup dài 10 trang.
- **Khả năng tự phục hồi (Self-heal):** Hỗ trợ tự khởi động lại khi ứng dụng gặp sự cố.

### 4. Bốn khái niệm cốt lõi cần nắm

- **Dockerfile:**
    - **Về mặt kỹ thuật:** Là một file văn bản (text file) không có đuôi, chứa một loạt các câu lệnh hướng dẫn (instructions) để Docker đọc và tự động thực thi nhằm tạo ra một Image.
    - **Đơn giản hơn nữa:** Dockerfile giống như một "tờ công thức" hay kịch bản (script) để tạo ra Image. Nó chỉ cho Docker biết từng bước phải làm gì: mượn hệ điều hành nào, copy đống "code" của chúng ta vào thư mục nào, rồi chạy lệnh gì (như `dotnet restore`, `npm install`...) để nén tất cả lại thành cái "bản đóng gói" cuối cùng. (Nói rõ hơn nữa ở mục sau)
- **Image:**
    - Là "bản thiết kế" hoặc bộ cài đặt. Nó định nghĩa môi trường và phần mềm có bên trong (Ví dụ: 1 Image chứa Ubuntu 16.04 + PHP 7.2 + MySQL). Image có thể được chia sẻ cho bất kỳ ai.
    - **Đơn giản hơn nữa:** Image giống như là 1 bản đóng gói code của chúng ta. Nghe có vẻ giống như folder sau khi mà chúng ta build xong (`dotnet build`, `npm run build`). *(Lưu ý một chút về thuật ngữ: "code" là đống chữ chúng ta viết ra thôi chưa có bấm run, bấm run thì nó thành "app").* Sự khác biệt chi tiết giữa Image và folder build sẽ được giải thích kỹ hơn ở phần sau.
- **Container:**
    - Là một thực thể (Instance || Object) đang chạy được tạo ra từ Image (giống như Object được khởi tạo từ Class trong lập trình). Từ 1 Image, bạn có thể chạy hàng trăm Container.
    - **Đơn giản hơn nữa:** Image là 1 bản đóng gói app của chúng ta, đóng gói lại xong thì cũng phải có lúc mở ra xài đúng không? Và sau khi mở phần đóng gói ra và bắt đầu xài ta thu được **Container** - chính là app của chúng ta đang chạy thực tế trên môi trường ảo hóa.
- **Volume:**
    - Hệ thống file của Container sẽ mất đi hoàn toàn nếu Container bị xóa hoặc khởi động lại. Volume là cơ chế chia sẻ bộ nhớ (mount) giữa máy thật và Container, giúp lưu trữ lại Database, Log, Ảnh... một cách vĩnh viễn dù cho Container có "chết" đi chăng nữa.

## 5. Phân tích Docker File và File Build của app (Lấy ví dụ thông qua app .NET)

Trước khi so sánh, chúng ta cần làm rõ một thuật ngữ cốt lõi mà lập trình viên nào cũng nghe mỗi ngày: **Build app**.

**Tại sao chúng ta phải "Build app"?**

- **Sự thật là:** Máy tính "ngốc" hơn chúng ta nghĩ. CPU của máy tính không hề biết đọc C#, Java, hay Javascript... Thứ duy nhất nó thực sự hiểu là dòng điện bật/tắt, tương ứng với **mã nhị phân (0 và 1)**.
- Những dòng code chúng ta gõ ra hàng ngày thực chất chỉ là "ngôn ngữ của người" – giúp anh em lập trình viên đọc hiểu logic và làm việc với nhau.
- Vì vậy, chúng ta cần trải qua quá trình **Compile (Biên dịch) và Build**. Quá trình này đóng vai trò như một *"ông phiên dịch viên"*, dịch toàn bộ đống chữ tiếng người kia sang dạng mã mà máy tính hoặc môi trường chạy có thể hiểu và thực thi được.

**Sản phẩm thu được là gì?** Sau khi "ông phiên dịch" làm việc xong, đống chữ của ta biến thành một ứng dụng thực sự, được gom gọn vào một thư mục (folder). Gọi chung là **File Build**.

- *Với .NET:* Chạy lệnh `dotnet publish`, ta thu được thư mục `publish` chứa các file `.dll` (mã đã biên dịch), file `.exe`...
- *Với Frontend/NodeJS:* Chạy lệnh `npm run build`, ta thu được thư mục `dist` hoặc `build` chứa các file Javascript/CSS đã được nén, gộp và tối ưu hóa sẵn sàng để chạy.

(Ở đây mình sẽ không nói quá sâu về 1 app .NET được build và run như thế nào, nếu bạn nào có nhu cầu tìm hiểu thì ib mình nhé.)

**Vấn đề của File Build (Nỗi đau)** Nghe thì có vẻ hoàn hảo, nhưng File Build mắc một "căn bệnh" chí mạng: **Tính dựa dẫm**. Nó giống như một con cá vừa được vớt ra khỏi ao. Nó đòi hỏi môi trường xung quanh phải có sẵn "nước" thì nó mới sống được. Bạn mang thư mục `publish` của .NET ném lên một con Server trắng tinh, Server sẽ báo lỗi ngay: *"Tôi không biết chạy file này, cài .NET Runtime vào đây cho tôi!"*. Tương tự, code NodeJS thì đòi hỏi Server phải cài sẵn Node.js.

**Sự khác biệt khi có Dockerfile** Đây là lúc **Dockerfile** chứng minh sức mạnh. Dockerfile không thay thế quá trình Build, mà nó **nâng cấp** quá trình đó lên mức hoàn hảo. Thay vì chỉ đưa cho khách hàng "con cá" (File Build) và bắt họ tự đi tìm "nước", Dockerfile là một tờ kịch bản tự động hóa thực hiện các bước:

1. Tạo ra một cái bể kính (Hệ điều hành Linux thu nhỏ).
2. Bơm nước vào bể (Cài đặt sẵn .NET Runtime hoặc NodeJS).
3. Thả con cá vào (Copy cái File Build vào trong bể).
4. Đóng nắp niêm phong lại thành một cục duy nhất (gọi là **Image**).

**=> Kết luận:** File Build chỉ là "phần xác" của ứng dụng, còn **Dockerfile** là công thức đóng gói cả "phần xác" lẫn "môi trường sống" vào chung một khối. Bạn bê khối (Image) này đi bất cứ máy nào, nó cũng chạy ngon lành 100%!. Câu chuyện con cá (file build - cá thiếu môi trường không thể sống được) và câu chuyện 1 bể cá hoàn chỉnh (dockerfile - có cá và cả môi trường).

Oker bây giờ mình sẽ tạo 1 file Dockerfile để đóng gói thử ứng dụng .NET của mình

1 File Dockerfile của 1 ứng dụng .NET được gen từ rider.

```docker
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["TetPee.Api/TetPee.Api.csproj", "TetPee.Api/"]
COPY ["TetPee.Service/TetPee.Service.csproj", "TetPee.Service/"]
COPY ["TetPee.Repository/TetPee.Repository.csproj", "TetPee.Repository/"]
RUN dotnet restore "TetPee.Api/TetPee.Api.csproj"
COPY . .
WORKDIR "/src/TetPee.Api"
RUN dotnet build "TetPee.Api.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "TetPee.Api.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "TetPee.Api.dll"]
```

### Giai đoạn 1: Chuẩn bị "bể cá" và "nước" (Tạo môi trường chạy rỗng)

Câu lệnh `FROM [mcr.microsoft.com/dotnet/aspnet:8.0](https://mcr.microsoft.com/dotnet/aspnet:8.0) AS base` chính là điểm khởi đầu (viên gạch nền móng) của file kịch bản Dockerfile.

Để dễ hiểu nhất, chúng ta hãy "mổ xẻ" câu lệnh này ra thành 4 phần nhé:

- **`FROM`**: Đây là từ khóa ra lệnh. Nó có nghĩa là: *"Hãy tải cho tôi một cái Image có sẵn ở trên mạng về đây để làm nền tảng"*. (Giống như việc bạn ra tiệm mua một cái bể cá làm sẵn thay vì tự đi mua kính về tự ráp).
- **`[mcr.microsoft.com/dotnet/aspnet](https://mcr.microsoft.com/dotnet/aspnet)`**: Đây là địa chỉ kho chứa và tên của cái Image mà bạn muốn lấy. Cụ thể ở đây, nó là kho chứa chính chủ của Microsoft (MCR - Microsoft Container Registry). Image này đã được Microsoft cài sẵn một hệ điều hành Linux thu nhỏ và **.NET Runtime** (công cụ chuyên dùng để *chạy* app, không chứa đồ nghề build app nên nó rất nhẹ).
- **`:8.0`**: Đây là Tag (phiên bản). Nó chỉ định rõ ràng là bạn muốn lấy môi trường .NET phiên bản 8.0. Nếu bạn code dự án bằng .NET 7, bạn sẽ phải đổi số này thành `:7.0`.
- **`AS base`**: Đây là hành động **đặt biệt danh**. Nó mang ý nghĩa: *"Từ giờ trở đi trong file kịch bản này, hãy gọi cái môi trường tôi vừa kéo về bằng một cái tên ngắn gọn là `base`"*. Nhờ việc đặt tên này, ở bước cuối cùng (Giai đoạn 4 vắt chanh bỏ vỏ), bạn mới có thể gọi lại nó bằng lệnh `FROM base AS final`.
- `AS base`: Đặt tên cho giai đoạn này là `base` để lát nữa gọi lại.
- `WORKDIR /app`: Tạo một thư mục gốc tên là `/app` bên trong container.
- `EXPOSE`: Đục 2 cái lỗ (cổng 8080 cho HTTP và 8081 cho HTTPS) để bên ngoài có thể gọi vào ứng dụng.

### Giai đoạn 2: Thuê "Thợ xây" và sơ chế nguyên liệu (Build)

- **Mục đích:** Mượn một cái Image "siêu to khổng lồ" mang tên `.NET SDK` (chứa đầy đủ búa kìm, cờ lê, mỏ lết) để chuyên phục vụ cho việc Compile code.
- **Chi tiết (Mẹo tối ưu tốc độ đỉnh cao của Docker):**
    - Đầu tiên, bạn cần biết file `.csproj` chính là tờ giấy ghi chú danh sách các thư viện (dependency) mà dự án của chúng ta sẽ xài.
    - **Câu hỏi đặt ra:** *Tại sao không `COPY` một phát toàn bộ code vào luôn cho lẹ, mà phải lóc cóc `COPY` từng cái file `.csproj` trước?*
    - **Bí mật nằm ở "Layer Caching":** Docker cực kỳ thông minh, nó có khả năng "nhớ" lại các bước đã làm. Nếu bạn sửa code (file `.cs`) mà **không** cài thêm thư viện mới, Docker sẽ nhận ra file `.csproj` không hề thay đổi. Nhờ vậy, nó sẽ dùng luôn kết quả cũ và **bỏ qua** bước tải thư viện cực kỳ tốn thời gian. Việc này giúp tốc độ build lại (re-build) diễn ra chớp nhoáng!
- **Giải thích luồng chạy của các lệnh:**
    - `RUN dotnet restore`: Cầm tờ giấy `.csproj` lên mạng, "vét" hết các thư viện cần thiết đem về máy.
    - `COPY . .`: Sau khi đã gom đủ đồ nghề thư viện, lúc này mới copy toàn bộ phần code thật (các file `.cs`, `appsettings.json`...) vào trong.
    - Nhãy vào dự án Entry - Dự án API (Chứa file Program): Và bắt đầu build app.
    - `RUN dotnet build`: Bắt đầu ra lệnh cho thợ xây dịch đống code nháp kia ra thành file nhị phân, rồi vứt tạm thành phẩm vào thư mục `/app/build`.

### Phân tích ý nghĩa 2 dấu chấm

- **Dấu `.` đầu tiên (Nguồn - Source):** Đại diện cho thư mục hiện tại ở **ngoài máy tính thật của bạn** (chính là nơi bạn đang mở terminal và gõ lệnh `docker build`).
- **Dấu `.` thứ hai (Đích đến - Destination):** Đại diện cho thư mục làm việc hiện tại **bên trong Container**. Cái thư mục này chính là nơi bạn đã chỉ định ở câu lệnh `WORKDIR` nằm ngay phía trên (ví dụ: `WORKDIR /src`).

**Dịch ra tiếng người:** _"Hãy bưng toàn bộ tất cả các file và thư mục đang có ở ngoài máy thật của tôi, ném hết vào trong cái thư mục `/src` của Container đi!"

### Giai đoạn 3: Đóng gói "Bản in hoàn chỉnh" (Publish)

```docker
FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "TetPee.Api.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false
```

- **Mục đích:** Kế thừa lại thành quả của anh Thợ xây ở giai đoạn 2 (`FROM build AS publish`), sau đó thực hiện chốt sổ.
- **Chi tiết:** Nó chạy lệnh `dotnet publish` để gom nhặt tất cả file `.dll`, thư viện, file tĩnh... nhét chung vào một cái hộp hoàn chỉnh và gọn gàng nhất tại thư mục `/app/publish`.
- Để thực sự mà mang app .NET lên production thì chúng ta cần phải thêm 1 bước ngoài dotnet build nữa là dotnet publish (Khúc này các bạn tự tìm hiểu nha dễ lắm)

### Giai đoạn 4: Vắt chanh bỏ vỏ - Lấy thành phẩm (Final)

Dockerfile

```
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "TetPee.Api.dll"]
```

- **Mục đích:** Đây là cú chốt hạ ăn tiền nhất của Multi-stage.
- **Chi tiết:**
    - `FROM base AS final`: Kéo cái "bể cá" siêu nhẹ ở **Giai đoạn 1** ra lại.
    - `COPY --from=publish /app/publish .`: **Phép thuật ở đây!** Nó thò tay sang bên **Giai đoạn 3** (`publish`), bốc đúng cái thư mục `/app/publish` (chứa sản phẩm cuối cùng) và ném vào cái bể cá siêu nhẹ này.
    - Sau hành động này, toàn bộ cái môi trường `.NET SDK` nặng cả Gigabyte cùng đống bừa bộn rác rưởi lúc build code ở giai đoạn 2 và 3 sẽ **bị vứt bỏ hoàn toàn**, không được đưa vào Image cuối.
    - `ENTRYPOINT`: Lệnh khai báo rằng: "Khi có ai đó bấm nút Start (chạy Container), hãy gõ lệnh `dotnet TetPee.Api.dll` để ứng dụng này chạy lên nhé!".

### Lỗi mà thông thường 1 người xài Docker NewBie sẽ không hiểu

### > Lỗi tư duy kinh điển của Docker Newbie: "Ảo giác đường thẳng"

Khi mới học Docker, 99% newbie đều mắc một lỗi tư duy: Các bạn nhìn file Dockerfile từ trên xuống dưới và nghĩ rằng *"À, nó đang chạy lần lượt từng dòng trên cùng một cái máy tính"*.

**Sự thật là KHÔNG PHẢI VẬY!**

Trong kỹ thuật Multi-stage build, cứ mỗi lần bạn gõ một chữ `FROM` mới (không kế thừa), Docker sẽ lập tức đập đi xây lại một **Căn phòng bê tông hoàn toàn mới, trống rỗng và cách ly 100%**.

Ví dụ: Bạn có tải 100GB thư viện ở chữ `FROM` đầu tiên, thì đến chữ `FROM` thứ hai, mọi thứ lại trở về con số 0. Hai căn phòng này hoàn toàn "mù" và không nhìn thấy đồ đạc của nhau.

### > Sự vi diệu của việc "Tái sử dụng căn phòng"

Đến đây, chắc chắn sẽ có bạn học viên tinh ý thắc mắc:

> _"Ủa anh ơi, ở Giai đoạn 2 (thằng thợ xây) mình đã tốn công chạy `dotnet restore` để tải một nùi thư viện về rồi. Tại sao sang bước `publish` mình không tận dụng luôn đi anh, tạo phòng mới làm gì cho mất công tải lại?"
> 

**Câu trả lời là: Chúc mừng em, em tư duy rất chuẩn! Và thực tế là CÓ, chúng ta ĐANG TÁI SỬ DỤNG nó!**

Hãy nhìn kỹ lại dòng code mở đầu của Giai đoạn 3:

```docker
FROM build AS publish
```

Phép thuật nằm ở chữ `build`. Thay vì gọi một cái môi trường mới toanh từ trên mạng về, chúng ta ra lệnh cho Docker: *"Ê, đừng có tạo phòng mới! Hãy mở đúng cái **Căn phòng `build`** lúc nãy (nơi đã tải sẵn thư viện và build sẵn nháp) ra đây. Tôi sẽ đổi tên bảng hiệu của phòng này thành `publish` và làm tiếp công đoạn đóng gói ở ngay trong này!"*

**Lợi ích khổng lồ của việc kế thừa này:**

1. **Không phải tải lại từ đầu:** Vì đang đứng trong chính căn phòng cũ, nên toàn bộ thư viện (`dotnet restore`) và đống code nháp (`dotnet build`) vẫn còn nguyên trên bàn làm việc.
2. **Tốc độ bàn thờ:** Lệnh `dotnet publish` lúc này không cần phải đi tải mạng hay dịch lại code nữa. Nó chỉ việc lăng xăng gom nhặt các file `.dll` đã có sẵn, đóng gói lại thành cái thùng `/app/publish` là xong! Nhanh chớp nhoáng!

### > Câu hỏi mở rộng: Vậy tại sao không viết gộp luôn vào Giai đoạn 2 cho rảnh nợ?

Nếu đã dùng chung một căn phòng, tại sao chúng ta không xóa luôn chữ `FROM build AS publish` đi, rồi mang cái lệnh `RUN dotnet publish...` vứt thẳng lên Giai đoạn 2? Code ngắn hơn mà vẫn chạy được cơ mà?

- **Sự thật là:** Bạn gộp lại nó VẪN CHẠY ngon lành.
- **Nhưng lý do người ta tách ra là vì:** Tính chuyên nghiệp và rõ ràng (Separation of Concerns).
    - Giai đoạn `build`: Chỉ chuyên tâm vào việc kéo thư viện và biên dịch thử xem code có lỗi không.
    - Giai đoạn `publish`: Chỉ chuyên tâm vào việc đóng gói sản phẩm để xuất xưởng.
    - Viết tách ra như vậy giúp file Dockerfile trở thành một kịch bản có cấu trúc rõ ràng, dễ đọc, dễ sửa và mang đậm phong cách của một Senior (Kỹ sư có kinh nghiệm)!

## Folder Structure của chúng ta qua từng giai đoạn.

Để bạn có cái nhìn "xuyên thấu" toàn tập vào file Dockerfile này, chúng ta sẽ cùng đi qua từng Căn phòng (Giai đoạn) và soi xem cây thư mục của chúng biến đổi như thế nào nhé!

---

### 1. Giai đoạn 1: `base` (Căn phòng Showroom)

Dockerfile

```docker
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
...
```

Lúc này, Docker vừa xây xong một căn phòng mới toanh, cài sẵn .NET Runtime và tạo ra một thư mục rỗng tên là `/app`.

**Cây thư mục lúc này:**

Plaintext

```
/ (Gốc của Container base)
│
└── app/       <-- (Trống rỗng, đang chờ đón thành phẩm)
```

---

### 2. Giai đoạn 2: `build` (Căn phòng Xưởng mộc)

Dockerfile

```
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
...
WORKDIR /src
COPY . .
RUN dotnet build "TetPee.Api.csproj" -c $BUILD_CONFIGURATION -o /app/build
```

Docker tạo một căn phòng **hoàn toàn mới**. Nó copy source code của bạn vào `/src` và ra lệnh build, ném kết quả tạm thời vào `/app/build`.

**Cây thư mục lúc này:**

Plaintext

```
/ (Gốc của Container build)
│
├── src/                          <-- (Nơi chứa toàn bộ Source Code)
│   ├── TetPee.Api/
│   │   ├── TetPee.Api.csproj
│   │   ├── Program.cs
│   │   └── obj/                  <-- Sinh ra khi chạy dotnet restore
│   ├── TetPee.Service/
│   └── TetPee.Repository/
│
└── app/
    └── build/                    <-- (Kho chứa kết quả build nháp)
        ├── TetPee.Api.dll
        ├── TetPee.Service.dll
        ├── TetPee.Repository.dll
        ├── appsettings.json
        └── TetPee.Api.pdb
```

---

### 3. Giai đoạn 3: `publish` (Vẫn ở Xưởng mộc, nhưng làm khâu đóng gói)

Dockerfile

```
FROM build AS publish
...
RUN dotnet publish "TetPee.Api.csproj" -c $BUILD_CONFIGURATION -o /app/publish
```

Giai đoạn này **dùng lại Căn phòng của Giai đoạn 2**. Nó thừa hưởng toàn bộ `/src` và `/app/build`. Sau đó nó chạy lệnh `publish` để lọc ra những file xịn nhất, gom chung với thư viện bên ngoài và ném vào một cái hộp mới tên là `/app/publish`.

**Cây thư mục lúc này:**

Plaintext

```
/ (Gốc của Container publish - Kế thừa từ build)
│
├── src/                          <-- (Vẫn còn nguyên source code)
│   └── ...
│
└── app/
    ├── build/                    <-- (Vẫn còn kết quả build nháp)
    │   └── ...
    │
    └── publish/                  <-- ✨ THÙNG THÀNH PHẨM XUẤT XƯỞNG
        ├── TetPee.Api.dll        <-- (Các file .dll cuối cùng)
        ├── TetPee.Service.dll
        ├── TetPee.Repository.dll
        ├── appsettings.json      <-- (File cấu hình)
        ├── Microsoft.EntityFrameworkCore.dll <-- (Thư viện ngoài gom vào)
        └── wwwroot/              <-- (Nếu có chứa file tĩnh như CSS, JS, Ảnh)
```

---

### 4. Giai đoạn 4: `final` (Chốt hạ tại Showroom)

Dockerfile

```
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "TetPee.Api.dll"]
```

Khúc này là **cú lừa ngoạn mục nhất**. Docker dập bỏ hoàn toàn căn phòng Xưởng mộc (Giai đoạn 2 và 3). Nó quay lại **Căn phòng Showroom ở Giai đoạn 1**, cho anh shipper chui sang phòng `publish`, bê ĐÚNG CÁI THÙNG `/app/publish` ném vào thư mục `/app` hiện tại.

**Cây thư mục CUỐI CÙNG trong Image:**

Plaintext

```
/ (Gốc của Container final)
│
└── app/                          <-- (Chính là thùng thành phẩm được copy sang)
    ├── TetPee.Api.dll
    ├── TetPee.Service.dll
    ├── TetPee.Repository.dll
    ├── appsettings.json
    ├── Microsoft.EntityFrameworkCore.dll
    └── wwwroot/

(TUYỆT ĐỐI KHÔNG CÒN THƯ MỤC /src HOẶC /app/build NỮA)
```

**💡 Tổng kết:** Nhờ cấu trúc vắt chanh bỏ vỏ này, Image cuối cùng mang đi chạy trên Server sẽ **siêu nhẹ** (vì mất đi thư mục `/src` nặng nề và các file nháp) và **siêu bảo mật** (vì hacker thò vào cũng chỉ thấy file `.dll` chứ không thấy được code C# gốc của bạn)!

Oker bây giờ chúng ta sẽ move qua giai đoạn tiếp theo.