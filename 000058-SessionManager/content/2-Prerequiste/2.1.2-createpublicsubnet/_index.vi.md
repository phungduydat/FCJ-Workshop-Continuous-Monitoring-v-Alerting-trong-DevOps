---
title : "Triển Khai Docker Image Lên AWS ECR Trên Linux "
date :  "`r Sys.Date()`" 
weight : 2
chapter : false
pre : " <b> 2.1.2 </b> "
---

## Mục Tiêu

Hướng dẫn cài đặt AWS CLI, cấu hình tài khoản AWS, tạo repository trên ECR, và đẩy Docker image từ máy local lên Amazon ECR.

---

## 1. Cài Đặt AWS CLI Trên Linux

Chạy các lệnh sau để cài đặt AWS CLI v2:

```bash
sudo apt update
sudo apt install unzip curl -y

# Tải AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Giải nén
unzip awscliv2.zip

# Cài đặt
sudo ./aws/install

# Kiểm tra phiên bản
aws --version
```

---

## 2. Cấu Hình AWS CLI

Sau khi cài đặt, chạy lệnh sau để cấu hình thông tin tài khoản:

```bash
aws configure
```

Nhập các thông tin sau:

* `AWS Access Key ID`: Nhập Access Key từ IAM
* `AWS Secret Access Key`: Nhập Secret Key từ IAM
* `Default region name`: `ap-northeast-1` (hoặc vùng khác như `us-east-1`, `ap-southeast-1`,...)
* `Default output format`: `json`

---

## 3. Tạo ECR Repository Trên AWS Console

1. Truy cập: [https://console.aws.amazon.com/ecr](https://console.aws.amazon.com/ecr)
2. Chọn **Repositories** → **Create repository**
3. Nhập tên repository: `webenglish`
4. Chọn **Private** và giữ các thiết lập mặc định
5. Nhấn **Create repository**

Sau khi tạo xong, bạn sẽ nhận được URI của repo, ví dụ:

```
466322313916.dkr.ecr.ap-northeast-1.amazonaws.com/webenglish
```

---

## 4. Xây Dựng Docker Image

Di chuyển đến thư mục chứa `Dockerfile` và chạy lệnh sau:

```bash
docker build -t webenglish-app .
```

---

## 5. Đăng Nhập Vào Amazon ECR

Sử dụng AWS CLI để đăng nhập vào ECR (bắt buộc trước khi push):

```bash
aws ecr get-login-password --region ap-northeast-1 | \
docker login --username AWS \
--password-stdin 466322313916.dkr.ecr.ap-northeast-1.amazonaws.com
```

---

## 6. Gắn Tag Docker Image Với ECR URI

```bash
docker tag webenglish-app:latest \
466322313916.dkr.ecr.ap-northeast-1.amazonaws.com/webenglish:webenglish-app
```

---

## 7. Push Docker Image Lên ECR

```bash
docker push \
466322313916.dkr.ecr.ap-northeast-1.amazonaws.com/webenglish:webenglish-app
```

---

## 8. Kiểm Tra Trên AWS Console

Sau khi push thành công:

* Quay lại AWS Console > ECR
* Vào repository `webenglish`
* Kiểm tra image `webenglish-app` đã được đẩy lên với tag `latest`

---

## 9. Lưu Ý Bổ Sung

* Đảm bảo Docker đã được cài đặt và đang chạy (`sudo systemctl start docker`)
* IAM user cần quyền `AmazonEC2ContainerRegistryFullAccess`
* Có thể dùng Amazon EC2 để triển khai container từ image này

---

## 10. Tài Liệu Tham Khảo

* [AWS CLI Install Guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html)
* [Amazon ECR Documentation](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html)
* [Docker CLI Docs](https://docs.docker.com/engine/reference/commandline/cli/)

---
