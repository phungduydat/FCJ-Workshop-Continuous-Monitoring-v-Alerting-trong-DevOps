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

> **Mục tiêu**: Hệ thống cảnh báo đa cấp, tự động phát hiện bất thường, gửi cảnh báo đến đúng người, và tự phản hồi nếu cần.

---

## 📌 1. Thiết Kế Quy Trình Cảnh Báo 3 Cấp

| Cấp | Tên                        | Người nhận       | Thời gian phản hồi   | Công cụ      |
| --- | -------------------------- | ---------------- | -------------------- | ------------ |
| 1   | Cảnh báo kỹ thuật (DevOps) | Nhóm DevOps      | ≤ 15 phút            | Email / SNS  |
| 2   | Cảnh báo gấp (On-call)     | Dev trực hotline | ≤ 5 phút             | SNS + Lambda |
| 3   | Cảnh báo quản lý (Manager) | Quản lý cấp cao  | Trong giờ hành chính | Email / SMS  |

---

## 🧪 2. Tạo SNS Topic và Subscriptions

### 2.1 Tạo các SNS Topic

```bash
aws sns create-topic --name WebEnglishAlert-Level1
aws sns create-topic --name WebEnglishAlert-Level2
aws sns create-topic --name WebEnglishAlert-Level3
```
![FWD](/images/5.fwd/1.jpg)

### 2.2 Đăng ký Email vào Topic (phải xác nhận)

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-northeast-1:<ACCOUNT_ID>:WebEnglishAlert-Level1 \
  --protocol email \
  --notification-endpoint devops@example.com

aws sns subscribe \
  --topic-arn arn:aws:sns:ap-northeast-1:<ACCOUNT_ID>:WebEnglishAlert-Level3 \
  --protocol email \
  --notification-endpoint ceo@example.com
```
![FWD](/images/5.fwd/2.jpg)

> ✅ **Kiểm tra email để xác nhận (Confirm Subscription)**.
![FWD](/images/5.fwd/3.jpg)
![FWD](/images/5.fwd/4.jpg)


---

## 📈 3. Tạo CloudWatch Alarm - Level 1

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name CPU-High-Level1 \
  --alarm-description "Cảnh báo CPU cao - cấp 1" \
  --metric-name cpu_usage_active \
  --namespace WebEnglishMetrics \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --actions-enabled \
  --alarm-actions arn:aws:sns:ap-northeast-1:<ACCOUNT_ID>:WebEnglishAlert-Level1
```
![FWD](/images/5.fwd/6.jpg)

---

## 🛠 4. Lambda Leo Thang

### 4.1 Tạo IAM Role cho Lambda

**trust-policy.json**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

```bash
aws iam create-role \
  --role-name LambdaAlertEscalatorRole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name LambdaAlertEscalatorRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess

aws iam attach-role-policy \
  --role-name LambdaAlertEscalatorRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSNSFullAccess
```
![FWD](/images/5.fwd/7.jpg)
![FWD](/images/5.fwd/8.jpg)


---

### 4.2 Mã Lambda: `alert_escalator.py`

```python
import boto3

def lambda_handler(event, context):
    cloudwatch = boto3.client('cloudwatch')
    sns = boto3.client('sns')

    alarm_name = "CPU-High-Level1"
    response = cloudwatch.describe_alarms(AlarmNames=[alarm_name])

    if response['MetricAlarms'] and response['MetricAlarms'][0]['StateValue'] == "ALARM":
        sns.publish(
            TopicArn="arn:aws:sns:ap-northeast-1:<ACCOUNT_ID>:WebEnglishAlert-Level2",
            Subject="🚨 Cảnh báo leo thang",
            Message=f"Alarm {alarm_name} vẫn chưa được xử lý. Đang leo thang lên cấp 2."
        )
```

---

### 4.3 Deploy Lambda Function

```bash
zip function.zip alert_escalator.py

aws lambda create-function \
  --function-name alertEscalator \
  --runtime python3.12 \
  --role arn:aws:iam::<ACCOUNT_ID>:role/LambdaAlertEscalatorRole \
  --handler alert_escalator.lambda_handler \
  --zip-file fileb://function.zip \
  --timeout 10
```

> ✅ Bạn có thể test thử bằng AWS Console → Lambda → Test
![FWD](/images/5.fwd/7.jpg)

---

## 📈 5. Tạo Alarm Gọi Lambda - Level 2

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name CPU-High-Level2 \
  --alarm-description "Leo thang cảnh báo CPU cao" \
  --metric-name cpu_usage_active \
  --namespace WebEnglishMetrics \
  --statistic Average \
  --period 600 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --actions-enabled \
  --alarm-actions arn:aws:lambda:ap-northeast-1:<ACCOUNT_ID>:function:alertEscalator
```

---

## 🔁 Tổng Quan Leo Thang

| Cấp | Alarm              | Điều kiện           | Hành động                                             |
| --- | ------------------ | ------------------- | ----------------------------------------------------- |
| 1   | CPU-High-Level1    | 1 × 5 phút > 80%    | Gửi SNS Level 1 → DevOps                              |
| 2   | CPU-High-Level2    | 2 × 10 phút > 80%   | Gọi Lambda → gửi SNS Level 2 (gọi người trực hotline) |
| 3   | (tùy chọn mở rộng) | Không xử lý sau 30p | Gửi SNS Level 3 → quản lý cấp cao qua email/sms       |

---

## 📊 6. CloudWatch Anomaly Detection (Tùy chọn nâng cao)

```bash
aws cloudwatch put-anomaly-detector \
  --namespace WebEnglishMetrics \
  --metric-name cpu_usage_active \
  --statistic Average \
  --dimensions Name=InstanceId,Value=i-xxxxxx
```
![FWD](/images/5.fwd/8.jpg)

---

## 🌀 7. Step Functions (Tùy chọn tự động phản hồi)

Sử dụng để:

* Scale ECS Service
* Reboot EC2 Instance
* Gửi chuỗi hành động khi phát hiện sự cố

---

## 🧪 8. Kiểm thử hệ thống

```bash
sudo yum install -y stress-ng
stress-ng --cpu 4 --timeout 600s
```
Bạn có thể gửi metric giả vào CloudWatch để test hệ thống hoạt động:

```bash
aws cloudwatch put-metric-data \
  --metric-name cpu_usage_active \
  --namespace WebEnglishMetrics \
  --value 90
```

Kiểm tra:

* SNS gửi đúng người?
* Lambda có được kích hoạt?
* Escalation có được thực hiện?
* Email xác nhận đã được nhấn chưa?

---

## 📘 9. Tổng Kết

✅ Hệ thống cảnh báo nhiều cấp hoạt động chuẩn:

* CloudWatch theo dõi & phát hiện bất thường
* SNS phân tầng cảnh báo đúng người
* Lambda phản ứng khi cảnh báo chưa được xử lý
* Mở rộng với Step Functions, PagerDuty, EC2 auto recovery...
![FWD](/images/5.fwd/9.jpg)
![FWD](/images/5.fwd/10.jpg)

---

## 🧩 Gợi ý nâng cao

* Dùng **CloudFormation / CDK** để mã hóa toàn bộ hạ tầng này
* Ghi log vào **CloudWatch Logs** để tracking escalation
* Tạo **Dead Letter Queue** cho Lambda nếu bị lỗi

---