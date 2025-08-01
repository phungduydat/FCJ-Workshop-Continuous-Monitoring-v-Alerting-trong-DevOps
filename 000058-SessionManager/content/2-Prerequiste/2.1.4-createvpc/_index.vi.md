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
sudo yum update -y

# Cài đặt Java OpenJDK 21
sudo amazon-linux-extras enable corretto8
sudo yum install -y java-21-amazon-corretto-devel

# Kiểm tra phiên bản đã cài đặt
java -version
```

---

## 🔧 2. Cài Đặt Maven

### 📥 Tải Maven


Tải xuống tệp nhị phân Maven mới nhất (phiên bản **3.9.11** tính đến tháng 7/2025):

```bash
sudo yum install java-11-amazon-corretto-devel
wget https://archive.apache.org/dist/maven/maven-3/3.9.1/binaries/apache-maven-3.9.1-bin.tar.gz
```

### 📦 Giải Nén Maven

```bash
sudo yum install java-11-amazon-corretto-devel
sudo mv apache-maven-3.9.1 /opt/
```

### ⚙️ Cấu Hình Biến Môi Trường

```bash
echo 'export M2_HOME=/opt/apache-maven-3.9.1' >> ~/.bashrc
echo 'export PATH=$PATH:$M2_HOME/bin' >> ~/.bashrc
source ~/.bashrc
```

### ✅ Kiểm Tra Maven

```bash
mvn -version
```

---

## 🐳 3. Cài Đặt Docker

### 🧹 Gỡ Docker Cũ (nếu có)

```bash
sudo dnf remove -y docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
```

### 📦 Cài Gói Phụ Thuộc

```bash
sudo dnf install -y dnf-plugins-core device-mapper-persistent-data lvm2
```

### 🗂️ Thêm Kho Docker CE

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo sed -i 's/$releasever/41/g' /etc/yum.repos.d/docker-ce.repo
```

### 🐳 Cài Docker 28.1.1

```bash
sudo dnf install -y docker-ce-28.1.1 docker-ce-cli-28.1.1 containerd.io docker-buildx-plugin docker-compose-plugin
```

Nếu không thấy phiên bản 28.1.1:

```bash
dnf list --showduplicates docker-ce
```

### 🚀 Khởi Động Docker

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

---

## 🧱 4. Cài Đặt Docker Compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

bash
sudo chmod +x /usr/local/bin/docker-compose
3. Tạo liên kết tượng trưng đến thư mục cli-plugins của Docker:
bash
sudo mkdir -p /usr/libexec/docker/cli-plugins
sudo ln -s /usr/local/bin/docker-compose /usr/libexec/docker/cli-plugins/docker-compose
---

## 📆 5. Cài Đặt Git & Clone Dự Án

```bash
sudo yum install -y git
git clone https://github.com/phungduydat/webenglish.git
cd webenglish
```

---

## 🚰 6. Build Spring Boot (Bỏ Qua Test)

```bash
mvn clean package -DskipTests
```

File jar sẽ nằm trong thư mục `target/`.

---

## 📄 7. Tạo File Dockerfile

```dockerfile
FROM openjdk:21-jdk
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

---
## 🧩 8. Tạo File docker-compose.yml

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
      - ./src/main/resources:/app/resources

volumes:
  mysql_data:
```

---

## ⚙️ 9. Cấu Hình `application.properties`

```properties
spring.datasource.url=${SPRING_DATASOURCE_URL}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD}
```

---

## 🧪 10. Build & Run Docker Compose

```bash
docker compose up --build
```

Kiểm tra:

```bash
docker ps
docker compose logs
```

---

**Tác giả:** Phùng Duy Đạt  
**GitHub:** [https://github.com/phungduydat/webenglish](https://github.com/phungduydat/webenglish)
