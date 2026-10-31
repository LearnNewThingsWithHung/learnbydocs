\---------------------Cấu hình các thứ-------------------------

\- lệnh build image từ dockerfile

docker build -f .\\TetPee.Api\\Dockerfile -t tetpee-api:1.0 .



\- lệnh tạo và chạy container từ 1 image

docker run --name <tetpee-api-container> -p 8080:8080 <7d415f190e04(id của images)>



\- image của postgres, cấm xóa nha

docker pull postgres:16-alpine



\- câu lệnh tạo một container từ images của postgres

docker run -d --name tetpee-postgres --network tetpee-network -e POSTGRES\_DB=tetpee-docker -e POSTGRES\_USER=postgres -e POSTGRES\_PASSWORD=MyStrongPassword123@ -v tetpee-postgres-data:/var/lib/postgresql/data postgres:16-alpine

\---------

docker run -d --name tetpee-postgres --network tetpee-network -p 5433:5432 -e POSTGRES\_DB=tetpee-docker -e POSTGRES\_USER=postgres -e POSTGRES\_PASSWORD=YOUR\_PASSWORD -v tetpee-postgres-data:/var/lib/postgresql/data postgres:16-alpine

\---------

docker run -d --name tetpee-api-container --network tetpee-network -p 8080:8080 -e ASPNETCORE\_ENVIRONMENT=Development -e ASPNETCORE\_HTTP\_PORTS=8080 -e "ConnectionStrings\_\_DefaultConnection=Host=tetpee-postgres;Port=5432;Database=tetpee-docker;Username=postgres;Password=YOUR\_PASSWORD" tetpee-api:1.0



\- câu lệnh tạo network

docker network create tetpee-network

\-> Hiểu đơn giản, mỗi container có một môi trường độc lập, mà app của bạn cần db để chạy -> cần connect 2 cái container (app - db) lại với nhau -> thì đưa bọn chúng vào chung một network



\-> Docker network cung cấp một DNS nội bộ, nên app có thể tìm PostgreSQL bằng tên của container thay vì phải nhớ IP



\- list network

docker network ls



\- xem thông tin của network

docker network inspect <tetpee-network(name)>



\- câu lệnh: đưa thằng nào đó vô chung 1 network

docker network connect tetpee-network tetpee-api-container



\-----------------------các thao tác bên trong một container

\- câu lệnh: nhảy vào bên trong một container đang chạy

docker exec -it tetpee-postgres psql -U postgres -d tetpee-docker 

&#x20;  -> di thằng vào postgres CLI

docker exec -it tetpee-api-container sh -> mở csl shell trong container



\- câu lệnh tạo file và xem log

$ echo "Dong log dau tien" > /tmp/demo.log

$ echo "Dong log thu hai" >> /tmp/demo.log

$ cat /tmp/demo.log

Dong log dau tien

Dong log thu hai



\- câu lệnh xóa container:

docker rm tetpee-api-container



\- câu lệnh add migration:

dotnet ef migrations add TenMigration --project TetPee.Repository --startup-project TetPee.Api --context AppDbContext







\- lệnh xem table

\\dt



\- xem cấu trúc của một bảng

\\d "Users"



\- Cách chia sẻ: Image + cấu hình chạy + dữ liệu cần thiết



\- xuất image thành file 

docker save -o tetpee-api.tar tetpee-api:1.0

\- người khác nhận rồi import

docker load -i tetpee-api.tar

\- tạo container từ image đó 



\---------------

compose.yml: đọc giúp tự cấu hình



dockerfile + src code + compose.	

\- câu lệnh push lên docker hub

docker tag tetpee-api:1.0 tanhung206/tetpee-api:1.0

=> gắn tag để dockerhub biết: push vào tài khoảng này, repo nào và phiên bản nào

tanhung206/tetpee-api:1.0

│          │          └─ Tag/phiên bản

│          └──────────── Tên repository

└─────────────────────── Tài khoản Docker Hub

docker tag tetpee-api:1.0 tanhung206/tetpee:api-1.0

docker tag postgres:16-alpine tanhung206/tetpee:postgres-16-alpine

docker push tanhung206/tetpee:api-1.0

docker push tanhung206/tetpee:postgres-16-alpine



\- cêu lệnh list tất cả các image của một repo trên remote

docker images tanhung206/tetpee







\---------------Các lệnh sau khi pull src code về



Flow tổng quát:

Chỉ cân git + Docker destop



Flow tổng quát:



Clone source

→ tạo .env

→ kiểm tra Compose

→ pull/build image

→ chạy container

→ chạy migration

→ kiểm tra

→ mở ứng dụng



Bước 1: Clone source code

git clone <GITHUB\_REPOSITORY\_URL>



Bước 2: cd vào prodject đó



Bước 3: đăng nhật registry nếu image private



Bước 4: Tạo file cấu hình .env

PowerShell:

Copy-Item .docker\\.env.sample .docker\\.env



Git Bash:

cp .docker/.env.sample .docker/.env



Bước 5: Điền cấu hình thật:

PowerShell:

notepad .docker\\.env



Bước 6: Kiểm tra cú pháp Compose

docker compose --env-file .docker/.env -f .docker/docker-compose.yml config -q



Bước 7A: Nếu compose dùng image có sẵn:

docker compose --env-file .docker/.env -f .docker/docker-compose.yml pull

docker compose --env-file .docker/.env -f .docker/docker-compose.yml up -d



Bước 7B: Nếu compose có buid: và muốn build source vừa clone 

docker compose --env-file .docker/.env -f .docker/docker-compose.yml up -d --buil



Cách này lấy source vừa clone, dùng Dockerfile build image rồi chạy container. Máy không cần cài .NET SDK vì SDK nằm trong Docker build stage.



Bước 8: Chạy database migration \*\*\*

docker compose --env-file .docker/.env -f .docker/docker-compose.yml run --rm migrate



Bước 9: Kiểm tra container

docker compose --env-file .docker/.env -f .docker/docker-compose.yml ps



Bước 10: Xem log

Toàn bộ hệ thống:

docker compose --env-file .docker/.env -f .docker/docker-compose.yml logs -f



Một service cụ thể:

docker compose --env-file .docker/.env -f .docker/docker-compose.yml logs -f api



Bước 11: Truy cập ứng dụng

http://localhost:8080/swagger/index.html













