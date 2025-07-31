---
title : "Automated Response"
date :  "`r Sys.Date()`" 
weight : 5 
chapter : false
pre : " <b> 5. </b> "
---
---

---------------------

# 🚨 Triển Khai Hệ Thống Cảnh Báo & Leo Thang Sự Cố Chuyên Nghiệp

> Mục tiêu: Thiết lập hệ thống giám sát cảnh báo theo cấp độ, có khả năng tự động phát hiện bất thường, gửi thông báo phù hợp theo mức độ nghiêm trọng và tự động phản ứng nếu cần.

---

## 📌 1. Thiết Kế Quy Trình Cảnh Báo 3 Cấp

| Cấp độ | Tên                         | Người nhận                   | Thời gian phản hồi | Công cụ      |
| ------ | --------------------------- | ---------------------------- | ------------------ | ------------ |
| 1      | Cảnh báo kỹ thuật (DevOps)  | Nhóm DevOps                  | ≤ 15 phút          | Email/SNS    |
| 2      | Cảnh báo gấp (On-call)      | Dev trực hotline / PagerDuty | ≤ 5 phút           | SNS + Lambda |
| 3      | Cảnh báo quản lý (Quản trị) | Quản lý cấp cao              | Giờ hành chính     | Email/SMS    |

---

## ✅ 2. Tạo SNS Topic Cho Mỗi Cấp

```bash
aws sns create-topic --name WebEnglishAlert-Level1
aws sns create-topic --name WebEnglishAlert-Level2
aws sns create-topic --name WebEnglishAlert-Level3
```

### Subscribing Người Nhận

```bash
# DevOps team
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-northeast-1:xxx:WebEnglishAlert-Level1 \
  --protocol email --notification-endpoint devops@example.com

# Quản lý cấp cao
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-northeast-1:xxx:WebEnglishAlert-Level3 \
  --protocol email --notification-endpoint ceo@example.com
```

---

## ⚙️ 3. Tạo Alarm CPU Với CloudWatch

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name CPU-High-Level1 \
  --metric-name cpu_usage_active \
  --namespace WebEnglishMetrics \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --period 300 \
  --statistic Average \
  --alarm-actions arn:aws:sns:ap-northeast-1:xxx:WebEnglishAlert-Level1
```

---

## ⚡ 4. Lambda Tự Động Leo Thang Nếu Không Xử Lý

### Bước 1: Tạo IAM Role Cho Lambda

Role cần quyền:

* `cloudwatch:DescribeAlarms`
* `sns:Publish`

### Bước 2: Code Lambda (`alertEscalator`)

```python
import boto3

def lambda_handler(event, context):
    cloudwatch = boto3.client('cloudwatch')
    sns = boto3.client('sns')

    alarm_name = "CPU-High-Level1"
    resp = cloudwatch.describe_alarms(AlarmNames=[alarm_name])
    alarm = resp['MetricAlarms'][0]

    if alarm['StateValue'] == "ALARM":
        sns.publish(
            TopicArn="arn:aws:sns:ap-northeast-1:xxx:WebEnglishAlert-Level2",
            Subject="🚨 Escalation Triggered",
            Message=f"Cảnh báo cấp 1 chưa được xử lý - {alarm_name}"
        )
```

### Bước 3: Gắn Lambda Vào Alarm Level 2

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name CPU-High-Level2 \
  --metric-name cpu_usage_active \
  --namespace WebEnglishMetrics \
  --threshold 80 \
  --evaluation-periods 2 \
  --period 600 \
  --statistic Average \
  --alarm-actions arn:aws:lambda:ap-northeast-1:xxx:function:alertEscalator
```

---

## 📲 5. Tích Hợp Với PagerDuty

### Bước 1: Tạo Integration Trong PagerDuty

* Vào **PagerDuty > Services > Add Service**
* Tạo integration kiểu **Amazon CloudWatch**
* Lấy **Webhook URL**

### Bước 2: Subscribe SNS Với Webhook PagerDuty

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-northeast-1:xxx:WebEnglishAlert-Level2 \
  --protocol https \
  --notification-endpoint https://events.pagerduty.com/integration/xyz/enqueue
```

---

## ⚙️ 6. Cấu Hình CloudWatch Anomaly Detection

```bash
aws cloudwatch put-anomaly-detector \
  --namespace WebEnglishMetrics \
  --metric-name cpu_usage_active \
  --statistic Average \
  --dimensions Name=InstanceId,Value=i-xxxxxx
```

---

## 🔀 7. Optional: AWS Step Functions Phản Hồi Tự Động

Sử dụng Step Functions để:

* Tự động scale service (ECS/Fargate)
* Gửi lệnh reboot EC2
* Gửi email cảnh báo theo luồng xử lý

---

## 🧪 8. Kiểm Thử

```bash
# Làm tăng CPU
sudo yum install -y stress-ng
stress-ng --cpu 4 --timeout 1200s
```

Kiểm tra:

* SNS gửi email/sms thành công
* Lambda kích hoạt khi chưa xử lý
* Escalation diễn ra đúng logic

---

## 📘 9. Tổng Kết

* Đảm bảo **cloudwatch alarms hoạt động tốt**
* **Lambda function** chạy định kỳ kiểm tra trạng thái
* SNS chia từng cấp cảnh báo rõ ràng
* Kết hợp PagerDuty cho cảnh báo ngoài giờ
* Đội ngũ phản hồi theo vai trò và cấp độ
