---
title : "Monitoring Implementation"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 3.1. </b> "
---
# Hướng Dẫn Thiết Lập Giám Sát Hạ Tầng WebEnglish Trên AWS Linux 2

Tài liệu này hướng dẫn chi tiết cách triển khai hệ thống giám sát hạ tầng WebEnglish trên **Amazon Linux 2**, bao gồm EC2, container Docker, ứng dụng Spring Boot và cơ sở dữ liệu MySQL bằng AWS CloudWatch.

---

## 1. Mục Tiêu

* Giám sát hiệu suất (CPU, RAM, Disk) trên EC2.
* Thu thập log ứng dụng Spring Boot và MySQL.
* Cảnh báo qua SNS.
* Hiển thị dashboard trực quan với CloudWatch.

**Công cụ chính:** AWS CloudWatch, CloudWatch Agent, Amazon SNS.

---

## 2. Chuẩn Bị

### EC2 Instance

* Amazon Linux 2
* Cài Docker, Docker Compose, Java (Spring Boot), MySQL

### Quyền IAM cần thiết

Attach IAM Role có:

* `CloudWatchAgentServerPolicy`
* `AmazonSSMManagedInstanceCore`
* Quyền ghi log và metric vào CloudWatch

### Cài AWS CLI (nếu chưa có)

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

---

## 3. Cài Đặt CloudWatch Agent Trên Amazon Linux 2

### Bước 1: Cài Đặt Agent

```bash
sudo yum install -y amazon-cloudwatch-agent
```

### Bước 2: Cấu hình Agent

Chạy wizard tạo file cấu hình:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

> File tạo sẽ lưu ở:

```
/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

### Mẫu File Cấu Hình:

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "logfile": "/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log"
  },
  "metrics": {
    "namespace": "WebEnglishMetrics",
    "metrics_collected": {
      "cpu": { "measurement": ["cpu_usage_active"], "totalcpu": true },
      "mem": { "measurement": ["mem_used_percent"] },
      "disk": {
        "measurement": ["disk_used_percent"],
        "resources": ["/"]
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/webenglish/app.log",
            "log_group_name": "WebEnglishLogs",
            "log_stream_name": "{instance_id}/app"
          },
          {
            "file_path": "/var/log/mysql/mysql.log",
            "log_group_name": "WebEnglishLogs",
            "log_stream_name": "{instance_id}/mysql"
          }
        ]
      }
    }
  }
}
```

### Bước 3: Khởi động Agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
```

Kiểm tra trạng thái:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

---

## 4. Thiết Lập Cảnh Báo (Alarm) với SNS

## ✅ Bước 1: Đăng ký Email vào SNS Topic

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-northeast-1:466322313916:WebEnglishAlerts \
  --protocol email \
  --notification-endpoint your_email@example.com
```

> 🔔 Thay `your_email@example.com` bằng email thật bạn muốn nhận cảnh báo.  
> 📬 Sau đó kiểm tra email và nhấp **"Confirm subscription"** để hoàn tất.

---

## ✅ Bước 2: Tạo CloudWatch Alarm giám sát CPU

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name HighCPUUsage \
  --metric-name cpu_usage_active \
  --namespace WebEnglishMetrics \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:ap-northeast-1:466322313916:WebEnglishAlerts
```

> 📌 Lưu ý:
> - `--namespace` phải trùng với namespace trong CloudWatch Agent.
> - `cpu_usage_active` là metric đã đẩy từ agent (nên kiểm tra bằng AWS Console > CloudWatch > Metrics).

---

## ✅ Bước 3: Gây tải CPU để kiểm tra Alarm

```bash
# Cài đặt stress nếu chưa có
sudo amazon-linux-extras install epel -y
sudo yum install stress -y

# Gây tải CPU: 2 core trong 5 phút
stress --cpu 2 --timeout 300
```

---

## ✅ Bước 4: Kiểm tra Alarm

- Truy cập: **AWS Console > CloudWatch > Alarms**
- Kiểm tra alarm `HighCPUUsage` có chuyển sang trạng thái `ALARM` khi CPU vượt ngưỡng.


## 6. Giám Sát MySQL Container

Tài liệu này hướng dẫn chi tiết cách triển khai hệ thống giám sát cho container MySQL bằng cách gửi số lượng truy vấn lên Amazon CloudWatch Metrics và thiết lập cronjob tự động.

---

## ✅ Bước 1: Cấu Hình MySQL Ghi Log Chậm *(tuỳ chọn)*

Nếu muốn giám sát truy vấn chậm, thêm cấu hình dưới đây:

### Sửa `docker-compose.yml`

```yaml
mysql:
  image: mysql:8.0
  container_name: mysql
  environment:
    MYSQL_ROOT_PASSWORD: 123456
    MYSQL_DATABASE: webenglish
  ports:
    - "3306:3306"
  volumes:
    - ./mysql.cnf:/etc/mysql/conf.d/mysql.cnf
```

### Nội dung `mysql.cnf`

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 1
```

Khởi động lại container:

```bash
docker restart mysql
```

---

## ✅ Bước 2: Tạo Bash Script Gửi Dữ Liệu Metric

### Tạo file script:

```bash
sudo nano /home/ec2-user/mysql_metrics.sh
```

### Dán nội dung sau:

```bash
#!/bin/bash

# Truy xuất số lượng truy vấn từ MySQL
QUERY_COUNT=$(docker exec mysql mysql -uroot -p123456 -e "SHOW GLOBAL STATUS LIKE 'Queries';" | grep Queries | awk '{print $2}')

# Gửi metric lên CloudWatch
aws cloudwatch put-metric-data \
  --metric-name MySQLQueries \
  --namespace WebEnglishMetrics \
  --value $QUERY_COUNT \
  --unit Count \
  --region ap-northeast-1
```

### Cấp quyền thực thi:

```bash
chmod +x /home/ec2-user/mysql_metrics.sh
```

---

## ✅ Bước 3: Thiết Lập Cronjob

```bash
crontab -e
```

Thêm dòng sau:

```bash
* * * * * /home/ec2-user/mysql_metrics.sh >> /var/log/mysql-metric.log 2>&1
```

---

## ✅ Bước 4: Kiểm Tra & Xác Nhận

### Kiểm tra log thực thi:

```bash
tail -f /var/log/mysql-metric.log
```

### Kiểm tra trên CloudWatch:

* Vào **CloudWatch > Metrics**
* Chọn **Namespace: `WebEnglishMetrics`**
* Kiểm tra `MySQLQueries`

---

## 🛠️ Tạo Alarm Cho MySQL Queries *(tuỳ chọn)*

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name HighMySQLQueries \
  --metric-name MySQLQueries \
  --namespace WebEnglishMetrics \
  --threshold 10000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --period 300 \
  --statistic Average \
  --alarm-actions arn:aws:sns:ap-northeast-1:466322313916:WebEnglishAlerts
```

---

## 📌 Ghi chú

* Đảm bảo container `mysql` đang chạy.
* Thay đổi mật khẩu và tên container nếu khác.
* Cronjob chạy mỗi phút — có thể điều chỉnh tần suất theo nhu cầu.
---

## 7. Tạo Dashboard CloudWatch

```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 12, "height": 6,
      "properties": {
        "metrics": [["WebEnglishMetrics", "cpu_usage_active"]],
        "region": "ap-northeast-1",
        "title": "CPU EC2 Usage"
      }
    },
    {
      "type": "log",
      "x": 12, "y": 0, "width": 12, "height": 6,
      "properties": {
        "query": "SOURCE 'WebEnglishLogs' | filter @logStream like 'app'",
        "region": "ap-northeast-1",
        "title": "App Logs"
      }
    }
  ]
}
```

---

## 8. Kiểm Thử và Xử Lý Sự Cố

```bash
sudo stress-ng --cpu 4 --timeout 300s
```

| Vấn đề           | Nguyên nhân       | Giải pháp                      |
| ---------------- | ----------------- | ------------------------------ |
| Agent không chạy | Thiếu quyền IAM   | Gán đúng IAM Role              |
| Không gửi log    | Sai đường dẫn log | Kiểm tra path trong config     |
| Không có metric  | Config sai        | Kiểm tra json và restart agent |

---

## 9. Gợi Ý Mở Rộng

* Tích hợp với X-Ray, AWS DevOps Guru.
* Sử dụng Lambda để phản ứng theo cảnh báo.
* Giám sát ECS/RDS nếu sử dụng thêm các dịch vụ đó.

---

## 📌 Ghi Chú

* Amazon Linux 2 dùng `yum`, file agent ở `/opt/aws/amazon-cloudwatch-agent/`.
* Kiểm tra log CloudWatch Agent:
  `/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log`
