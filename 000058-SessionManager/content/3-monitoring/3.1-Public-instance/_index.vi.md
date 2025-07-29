---
title : "Monitoring Implementation"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 3.1. </b> "
---

# Hướng Dẫn Thiết Lập Giám Sát Hạ Tầng WebEnglish

Tài liệu này hướng dẫn chi tiết cách triển khai hệ thống giám sát hạ tầng WebEnglish trên AWS, bao gồm giám sát Amazon EC2, container Docker, ứng dụng Spring Boot và cơ sở dữ liệu MySQL bằng AWS CloudWatch.

---

## 1. Mục Tiêu

* Theo dõi hiệu suất hệ thống (CPU, bộ nhớ, đĩa) của EC2 và container Docker.
* Thu thập nhật ký từ ứng dụng Spring Boot và MySQL.
* Thiết lập cảnh báo theo thời gian thực.
* Xây dựng dashboard trực quan hỗ trợ nhóm DevOps.

**Công cụ chính:** AWS CloudWatch, CloudWatch Agent, Amazon SNS.

---

## 2. Yêu Cầu Chuẩn Bị

### Hệ Thống:

* AWS Account với quyền IAM phù hợp.
* EC2 instance (Amazon Linux 2/Ubuntu, cỡ t2.medium), cài Docker và Spring Boot.
* Docker Compose triển khai Spring Boot + MySQL.

### Quyền IAM:

* EC2: `CloudWatchAgentServerPolicy`, `AmazonSSMManagedInstanceCore`.
* CloudWatch: `cloudwatch:PutMetricData`, quyền ghi log.

### Công Cụ:

* AWS CLI, AWS CDK (tuỳ chọn).
* Kiến thức cơ bản về EC2, Docker, CloudWatch.

### Cài Đặt AWS CLI:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

---

## 3. Triển Khai Chi Tiết

### 3.1. Cài CloudWatch Agent Trên EC2

#### Bước 1: Cài Đặt

```bash
# Amazon Linux 2
sudo yum install amazon-cloudwatch-agent -y

# Ubuntu
sudo apt-get install amazon-cloudwatch-agent -y
```

Kiểm tra:

```bash
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -v
```

#### Bước 2: Tạo Tệp Cấu Hình

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

#### Cấu Hình Mẫu `/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json`

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "logfile": "/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log"
  },
  "metrics": {
    "namespace": "WebEnglishMetrics",
    "metrics_collected": {
      "cpu": { "measurement": ["cpu_usage_active", "cpu_usage_idle"], "totalcpu": true },
      "mem": { "measurement": ["mem_used_percent", "mem_total"] },
      "disk": { "measurement": ["disk_used_percent"], "resources": ["/"] }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          { "file_path": "/var/log/webenglish/app.log", "log_group_name": "WebEnglishLogs", "log_stream_name": "{instance_id}/app.log" },
          { "file_path": "/var/log/mysql/mysql.log", "log_group_name": "WebEnglishLogs", "log_stream_name": "{instance_id}/mysql.log" }
        ]
      }
    }
  }
}
```

#### Khởi động Agent:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
```

---

### 3.2. Thiết Lập CloudWatch Alarms

#### Tạo SNS Topic:

* SNS > Create Topic: `WebEnglishAlerts`
* Thêm email DevOps team để nhận thông báo.

#### CLI Tạo Alarm:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name HighCPUUsage \
  --metric-name cpu_usage_active \
  --namespace WebEnglishMetrics \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --period 300 \
  --statistic Average \
  --alarm-actions arn:aws:sns:<region>:<account-id>:WebEnglishAlerts
```

---

### 3.3. Giám Sát Spring Boot

#### Pom Dependency:

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-cloudwatch</artifactId>
  <version>1.9.0</version>
</dependency>
```

#### Cấu hình `application.properties`

```properties
management.metrics.export.cloudwatch.namespace=WebEnglishAppMetrics
management.metrics.export.cloudwatch.step=1m
management.endpoints.web.exposure.include=metrics
```

#### Ví Dụ Java:

```java
@GetMapping("/example")
public String example() {
  meterRegistry.timer("webenglish.request.latency").record(() -> {
    Thread.sleep(100);
  });
  return "Hello WebEnglish";
}
```

---

### 3.4. Giám Sát MySQL Container

#### Cấu Hình `mysql.cnf`

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 1
```

#### Gửi Metric MySQL:

```bash
#!/bin/bash
QUERY_COUNT=$(mysql -uroot -proot -e "SHOW GLOBAL STATUS LIKE 'Queries';" | grep Queries | awk '{print $2}')
aws cloudwatch put-metric-data \
  --metric-name MySQLQueries \
  --namespace WebEnglishMetrics \
  --value $QUERY_COUNT \
  --unit Count
```

#### Cron Job:

```bash
crontab -e
* * * * * /path/to/mysql_metrics.sh
```

---

### 3.5. Tạo CloudWatch Dashboard

#### Cấu hình CLI mẫu:

```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 12, "height": 6,
      "properties": {
        "metrics": [["WebEnglishMetrics", "cpu_usage_active"], ["WebEnglishMetrics", "mem_used_percent"]],
        "region": "us-east-1",
        "title": "Hiệu suất EC2",
        "view": "timeSeries"
      }
    },
    {
      "type": "log",
      "x": 12, "y": 0, "width": 12, "height": 6,
      "properties": {
        "query": "SOURCE 'WebEnglishLogs' | fields @message | filter @logStream like 'app.log'",
        "region": "us-east-1",
        "title": "Nhật ký Ứng dụng"
      }
    }
  ]
}
```

---

## 4. Kiểm Thử

* **Kiểm thử đơn vị:** Kiểm tra CloudWatch Agent và chỉ số ứng dụng.
* **Kiểm thử tích hợp:** Giám sát cảnh báo SNS, log xuất hiện.
* **Hiệu suất:** `stress-ng --cpu 4 --timeout 600s`
* **Mô phỏng lỗi:** tạo truy vấn chậm hoặc lỗi 500 từ Spring Boot.

---

## 5. Thực Tiễn Tốt Nhất

* Nhật ký: Giới hạn lưu trữ 7–14 ngày.
* Chỉ số: Gửi mỗi 60 giây để tối ưu chi phí.
* IAM: Chỉ cấp quyền cần thiết.
* Cảnh báo: Điều chỉnh theo chu kỳ đánh giá thực tế.
* Runbook: Ghi lại hướng xử lý lỗi phổ biến.

---

## 6. Xử Lý Sự Cố

| Vấn đề             | Nguyên nhân            | Giải pháp                      |
| ------------------ | ---------------------- | ------------------------------ |
| Không thấy chỉ số  | Agent lỗi, thiếu quyền | Kiểm tra IAM và cấu hình Agent |
| Không có log       | Sai đường dẫn          | Kiểm tra đường dẫn tệp log     |
| Cảnh báo không gửi | Cấu hình SNS sai       | Kiểm tra ARN và ngưỡng         |
| Chi phí cao        | Lưu log quá lâu        | Điều chỉnh retention log       |

---

## 7. Bước Tiếp Theo

* Tích hợp AWS X-Ray để theo dõi request.
* Kích hoạt AWS DevOps Guru.
* Sử dụng AWS Lambda phản ứng tự động.
* Mở rộng giám sát ECS/RDS nếu cần.
