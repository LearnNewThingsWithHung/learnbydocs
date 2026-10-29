# Docker

**Docker** là một dự án mã nguồn mở, cho phép tự động hóa việc triển khai ứng dụng bên trong các **container**, đồng thời đóng gói những thành phần cần thiết để ứng dụng hoạt động. Docker cung cấp công cụ **CLI (Command Line Interface)** để quản lý vòng đời của container. Nhờ đó, việc phát triển, triển khai và bảo trì ứng dụng thuận tiện hơn.

Docker khác với **máy ảo (virtual machine)**:

- **Máy ảo:** Chạy một hệ điều hành riêng, cùng những thành phần cần thiết của hệ điều hành đó.
- **Docker container:** Cung cấp môi trường cách ly cho ứng dụng, thường chỉ gồm các thành phần cần thiết để chạy ứng dụng. Các container dùng chung **kernel** của hệ điều hành phía máy chủ (hoặc kernel của môi trường Linux VM mà Docker sử dụng).

Một số ưu điểm của Docker container so với máy ảo:

- **Tạo và xóa nhanh:** Container thường khởi tạo nhanh và dùng ít tài nguyên hơn máy ảo do không phải khởi động một hệ điều hành đầy đủ cho từng container.
- **Gọn nhẹ:** Thông thường có thể chạy nhiều container hơn máy ảo trên cùng một máy host, tùy cấu hình và khối lượng công việc.

## Một số lệnh Docker

### Kiểm tra thông tin và liệt kê

**`docker --version`**  
Kiểm tra phiên bản Docker.

**`docker info`**  
Xem thông tin hệ thống Docker.

**`docker images -a`**  
Liệt kê các image, bao gồm image trung gian.

**`docker pull nameimage:tag`**  
Tải một image từ registry (ví dụ Docker Hub).

**`docker ps`**  
Liệt kê các container đang chạy.

**`docker ps -a`**  
Liệt kê tất cả container, kể cả container đã dừng.

**`docker container ls -a`**  
Liệt kê tất cả container (tương đương `docker ps -a`).

---

### Tạo và chạy container

**`docker run -it --name nameyourcontainer -h "nameyourhost" image_id`**  
Tạo và chạy container từ image `image_id` (hoặc tên image). Trong đó `--name` đặt tên container, `-h` đặt hostname và `-it` dùng để tương tác qua terminal.

Một số tham số có thể thêm khi tạo container:

**`-v path-in-host:path-in-container`**  
Ánh xạ đường dẫn trên máy host vào container (bind mount khi dùng đường dẫn host).

**`--volumes-from other-container-name`**  
Gắn các volume/mount từ container khác vào container hiện tại.

**`-p public-port:target-port`**  
Ánh xạ cổng trên máy host (`public-port`) tới cổng bên trong container (`target-port`).

**`--restart=always`**  
Đặt chính sách tự khởi động lại container theo chính sách `always` của Docker.

**`docker container attach containerid`**  
Gắn terminal hiện tại vào luồng đầu vào/đầu ra chính của container đang chạy (không tạo shell mới).

**`docker exec -it containerid command`**  
Chạy một lệnh `command` bên trong container đang hoạt động.

**`docker stop containerid`**  
Dừng container.

**`docker start -i containerid`**  
Khởi động một container đã dừng, đồng thời gắn vào luồng đầu vào của container.

**`docker restart containerid`**  
Khởi động lại container.

**`docker rm containerid`**  
Xóa container đã dừng (mặc định không xóa container đang chạy nếu không dùng tùy chọn ép buộc).

**`Ctrl + P`, sau đó `Ctrl + Q`**  
Tách khỏi phiên terminal đã attach mà không dừng container (detach sequence mặc định).

---

### Image, network và theo dõi container

**`docker commit containerid imagename:imageversion`**  
Tạo image mới từ các thay đổi trong hệ thống tệp và cấu hình của container. Không bắt buộc container phải dừng; dữ liệu trong volume không được đưa vào image.

**`docker save --output myimage.tar myimage_id`**  
Lưu image thành tệp `.tar`.

**`docker load -i myimage.tar`**  
Nạp image từ tệp `.tar` vào Docker.

**`docker tag image_id imagename:version`**  
Gắn thêm tên và tag cho image (không thay đổi nội dung image).

**`docker network ls`**  
Liệt kê các network Docker.

**`docker network create --driver bridge name-network`**  
Tạo một network kiểu `bridge` có tên `name-network`.

**`docker network connect name-network name-container`**  
Kết nối container vào network `name-network`.

**`docker inspect name_or_id_of_image_container`**  
Xem thông tin chi tiết của image hoặc container.

**`docker history name_or_id_of_image`**  
Xem lịch sử các layer của image.

**`docker diff container-name-or-id`**  
Xem các thay đổi trong filesystem của container so với image gốc (không bao gồm nội dung volume).

**`docker logs -f container-name-or-id`**  
Xem log container và tiếp tục theo dõi các dòng log mới.

**`docker stats container-name-or-id`**  
Xem thống kê sử dụng tài nguyên của container (CPU, RAM, network, I/O...).
