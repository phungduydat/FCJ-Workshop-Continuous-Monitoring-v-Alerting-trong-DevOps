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

### ✅ Bước 1: Cài đặt CloudWatch Agent

```bash
cd /tmp
curl -O https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
sudo rpm -U ./amazon-cloudwatch-agent.rpm
```

### ✅ Bước 2: Tạo file cấu hình thủ công

```bash
sudo nano /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

#### Nội dung mẫu:

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "logfile": "/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log",
    "run_as_user": "root"
  },
  "metrics": {
    "namespace": "WebEnglishMetrics",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}",
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}"
    },
    "aggregation_dimensions": [["InstanceId"]],
    "metrics_collected": {
      "cpu": {
        "measurement": [
          "cpu_usage_user",
          "cpu_usage_system",
          "cpu_usage_idle"
        ],
        "metrics_collection_interval": 60,
        "totalcpu": true
      },
      "mem": {
        "measurement": [
          "mem_used_percent"
        ],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          "used_percent"
        ],
        "metrics_collection_interval": 60,
        "resources": ["*"]
      },
      "diskio": {
        "measurement": [
          "io_time"
        ],
        "metrics_collection_interval": 60
      },
      "swap": {
        "measurement": [
          "swap_used_percent"
        ],
        "metrics_collection_interval": 60
      }
    }
  }
}
```

### ✅ Bước 3: Khởi động agent với cấu hình vừa tạo

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s
```

### ✅ Bước 4: Kiểm tra trạng thái agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
```

---

## 4. Thiết Lập Cảnh Báo (Alarm) với SNS

### ✅ Bước 1: Tạo SNS Topic và Đăng Ký Email

```bash
aws sns create-topic \
  --name WebEnglishAlerts \
  --region ap-northeast-1
```

**Output mẫu:**

```json
{
  "TopicArn": "arn:aws:sns:ap-northeast-1:466322313916:WebEnglishAlerts"
}
```

Đăng ký email:

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-northeast-1:466322313916:WebEnglishAlerts \
  --protocol email \
  --notification-endpoint duydatphung7@gmail.com \
  --region ap-northeast-1
```

📬 Kiểm tra email và nhấn xác nhận để hoàn tất.

### ✅ Bước 2: Tạo CloudWatch Alarm giám sát CPU

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name HighCPUUsage \
  --metric-name cpu_usage_user \
  --namespace WebEnglishMetrics \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:ap-northeast-1:466322313916:WebEnglishAlerts \
  --region ap-northeast-1
```

### ✅ Bước 3: Gây tải CPU để kiểm tra Alarm

```bash
sudo amazon-linux-extras install epel -y
sudo yum install stress-ng -y

stress-ng --cpu 2 --timeout 300s
```

---

## 5. Giám Sát MySQL Container

### ✅ Bước 1: Cấu Hình MySQL Ghi Log Chậm *(tuỳ chọn)*

**docker-compose.yml:**

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

**mysql.cnf:**

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 1
```

```bash
docker restart mysql
```

### ✅ Bước 2: Tạo Bash Script Gửi Dữ Liệu Metric

```bash
sudo nano /home/ec2-user/mysql_metrics.sh
```

```bash
#!/bin/bash
QUERY_COUNT=$(docker exec mysql mysql -uroot -p123456 -e "SHOW GLOBAL STATUS LIKE 'Queries';" | grep Queries | awk '{print $2}')

aws cloudwatch put-metric-data \
  --metric-name MySQLQueries \
  --namespace WebEnglishMetrics \
  --value $QUERY_COUNT \
  --unit Count \
  --region ap-northeast-1
```

```bash
chmod +x /home/ec2-user/mysql_metrics.sh
```

### ✅ Bước 3: Thiết Lập Cronjob

```bash
crontab -e
```

Thêm dòng sau:

```bash
* * * * * /home/ec2-user/mysql_metrics.sh >> /var/log/mysql-metric.log 2>&1
```

### ✅ Bước 4: Tạo Alarm cho MySQL Queries *(tuỳ chọn)*

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
  --unit Count \
  --alarm-actions arn:aws:sns:ap-northeast-1:466322313916:WebEnglishAlerts \
  --region ap-northeast-1
```

---

## 6. Tạo Dashboard CloudWatch

**webenglish-dashboard.json**:

```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 12, "height": 6,
      "properties": {
        "metrics": [["WebEnglishMetrics", "cpu_usage_user"]],
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

```bash
aws cloudwatch put-dashboard \
  --dashboard-name WebEnglishDashboard \
  --dashboard-body file://webenglish-dashboard.json \
  --region ap-northeast-1
```

---

## 7. Kiểm Thử và Xử Lý Sự Cố

```bash
sudo stress-ng --cpu 4 --timeout 300s
```

| Vấn đề           | Nguyên nhân       | Giải pháp                     |
| ---------------- | ----------------- | ----------------------------- |
| Agent không chạy | Thiếu quyền IAM   | Gán đúng IAM Role             |
| Không gửi log    | Sai đường dẫn log | Kiểm tra path trong config    |
| Không có metric  | Config sai        | Kiểm tra JSON & restart agent |

---

## 8. Gợi Ý Mở Rộng

* Tích hợp với X-Ray, AWS DevOps Guru
* Sử dụng Lambda để phản ứng theo cảnh báo
* Giám sát ECS/RDS nếu sử dụng thêm dịch vụ

---

## 📌 Ghi Chú

* Amazon Linux 2 dùng `yum`, file agent ở `/opt/aws/amazon-cloudwatch-agent/`
* Kiểm tra log CloudWatch Agent tại:
  `/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log`
