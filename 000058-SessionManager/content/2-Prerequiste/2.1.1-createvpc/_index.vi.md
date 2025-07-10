---
title : " Chuẩn Bị Môi Trường Tạo Docker Image Cho Dự Án Spring Boot"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 2.1.1 </b> "
---

---

---

## 🔧 1. Cài Đặt Java (OpenJDK 21)

Để chạy được ứng dụng Spring Boot và build ra file `.jar`, bạn cần Java JDK 21.

```bash
# Cập nhật danh sách gói phần mềm và nâng cấp hệ thống
sudo apt update && sudo apt upgrade -y

# Cài đặt Java OpenJDK 21
sudo apt install -y openjdk-21-jdk

# Kiểm tra phiên bản đã cài đặt
java -version
```

---

## 🔧 2. Cài Đặt Maven

Maven là công cụ build để biên dịch và đóng gói ứng dụng Spring Boot.

```bash
# Cài đặt Maven
sudo apt install -y maven

# Kiểm tra phiên bản Maven để đảm bảo cài đặt thành công
mvn -version
```

---

## 🐳 3. Cài Đặt Docker (Docker Engine >= 19.03)

Docker sẽ dùng để tạo container cho ứng dụng Spring Boot. Phiên bản từ 19.03 trở lên mới hỗ trợ Compose file 3.8.

### Bước 1: Cập nhật hệ thống và cài đặt các gói phụ thuộc

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
```

### Bước 2: Thêm GPG key của Docker

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

### Bước 3: Thêm Docker repository chính thức từ Docker

```bash
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable"
```

### Bước 4: Cài đặt Docker Engine

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```

### Bước 5: Kiểm tra phiên bản Docker

```bash
docker --version
```

---

## 🧱 4. Cài Đặt Docker Compose (Hỗ Trợ Docker Compose Version 3.8)

Docker Compose giúp chạy nhiều container theo cấu hình YAML (ví dụ `docker-compose.yml`).

```bash
# Tải docker-compose bản 2.27.0 phù hợp với cấu hình hiện tại
sudo curl -L "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Cấp quyền thực thi cho docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Kiểm tra phiên bản
docker-compose --version
```

---

## 📆 5. Cài Đặt Git & Clone Dự Án Spring Boot

Git dùng để tải mã nguồn dự án về máy từ GitHub.

```bash
# Cài đặt Git nếu chưa có
sudo apt-get install -y git

# Clone dự án Spring Boot từ GitHub
git clone https://github.com/phungduydat/webenglish.git

# Di chuyển vào thư mục dự án
cd webenglish
```

---

## 🚰 6. Build Dự Án Spring Boot (Bỏ Qua Test)

```bash
# Build và đóng gói file .jar mà không chạy test
mvn clean package -DskipTests
```

Sau khi thực hiện, file jar sẽ nằm trong thư mục `target/`.

---

## 📄 7. Tạo File Dockerfile

Tạo file `Dockerfile` để chỉ định cách tạo Docker Image từ file jar.

```dockerfile
# Dùng image chính thức của Java 21
FROM openjdk:21-jdk

# Khai báo tên file jar
ARG JAR_FILE=target/*.jar

# Copy file jar vào image
COPY ${JAR_FILE} app.jar

# Thiết lập lệnh chạy ứng dụng khi container khởi động
ENTRYPOINT ["java","-jar","/app.jar"]
```

---

## 🧩 8. Tạo File `docker-compose.yml`

```yaml
services:
  mysql:
    image: mysql:8.0
    container_name: mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: webenglish
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  app:
    build: .
    container_name: webenglish-app
    ports:
      - "8080:8090"
    depends_on:
      - mysql
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/webenglish
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: 123456
    volumes:
      - mysql_data:

volumes:
  mysql_data:
```

**Lưu ý:** file YAML cần đúng thụt lề (2 dấu cách mỗi cấp).

---

## ⚙️ 9. Cấu Hình `application.properties`

Trong file `src/main/resources/application.properties`, chỉnh lại cấu hình datasource:

```properties
spring.datasource.url=${SPRING_DATASOURCE_URL}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD}
```

---

## 🧪 10. Build và Khởi Chạy Bằng Docker Compose

Sau khi đã tạo `Dockerfile`, `docker-compose.yml` và cấu hình `application.properties`, bạn có thể khởi chạy toàn bộ ứng dụng bằng Docker Compose.

```bash
# Build lại image và khởi chạy các service
docker compose up --build
```

Sau khi chạy, Docker sẽ:

* Tải image MySQL (nếu chưa có)
* Build image từ source code Spring Boot
* Khởi tạo hai container: `mysql` và `webenglish-app`
* Kết nối chúng theo định nghĩa trong `docker-compose.yml`

### ✅ Kiểm Tra Các Container Đang Chạy

```bash
# Danh sách các container đang chạy
docker ps
```

Đảm bảo bạn thấy hai container `mysql` và `webenglish-app` trong danh sách. Nếu có lỗi, kiểm tra log:

```bash
# Xem log từ tất cả service
docker compose logs
```

---
**Tác giả:** Phùng Duy Đạt
**GitHub:** [https://github.com/phungduydat/webenglish](https://github.com/phungduydat/webenglish)
