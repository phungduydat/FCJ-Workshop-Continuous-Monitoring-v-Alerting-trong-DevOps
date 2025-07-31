---
title : "Cập nhật IAM Role"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 4.1 </b> "
---
---

title: "Triển khai Tính năng Cảnh Báo và Phản Hồi Tự Động cho WebEnglish"
date: "`r Sys.Date()`"
weight: 8
chapter: false
pre: "<b> 3.4 </b>"
-------------------

# 🚨 Triển khai Cảnh Báo và Phản Hồi Tự Động với CloudWatch, SNS và Lambda

Tài liệu này hướng dẫn chi tiết cách tích hợp Amazon CloudWatch, SNS, Lambda, và (tuỳ chọn) DevOps Guru để phát hiện bất thường và tự động phản ứng trong hệ thống WebEnglish.

---

## 🎯 Mục Tiêu

* Phát hiện bất thường hiệu suất EC2/ECS.
* Gửi cảnh báo qua email và kích hoạt Lambda phản ứng.
* Scale ECS hoặc khởi động lại service tự động.

---

## ✅ 1. Tạo CloudWatch Anomaly Detector

```bash
aws cloudwatch put-anomaly-detector \
  --namespace WebEnglishMetrics \
  --metric-name cpu_usage_active \
  --statistic Average
```

---

## ✅ 2. Tạo SNS Topic và Đăng ký Email

```bash
aws sns create-topic --name WebEnglishAlerts

aws sns subscribe \
  --topic-arn arn:aws:sns:ap-northeast-1:466322313916:WebEnglishAlerts \
  --protocol email \
  --notification-endpoint your-email@example.com
```

> Xác nhận email để kích hoạt subscription.

---

## ✅ 3. Tạo CloudWatch Alarm từ Anomaly Detector

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name CPUAnomalyAlarm \
  --metric-name cpu_usage_active \
  --namespace WebEnglishMetrics \
  --statistic Average \
  --comparison-operator GreaterThanUpperThreshold \
  --threshold 0 \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --treat-missing-data notBreaching \
  --anomaly-detection-bands \
  --alarm-actions arn:aws:sns:ap-northeast-1:466322313916:WebEnglishAlerts
```

---

## ✅ 4. Tạo Lambda để Scale ECS

### 📄 Code Python `scale_ecs_service.py`

```python
import boto3

def lambda_handler(event, context):
    ecs = boto3.client('ecs')
    response = ecs.update_service(
        cluster='MyCluster',
        service='WebenglishService',
        desiredCount=2
    )
    print("Scaling triggered:", response)
    return response
```

### 🔐 IAM Role Cho Lambda

Cần quyền `ecs:UpdateService`, `logs:*`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["ecs:UpdateService", "ecs:DescribeServices"], "Resource": "*" },
    { "Effect": "Allow", "Action": "logs:*", "Resource": "*" }
  ]
}
```

---

## ✅ 5. Gắn Lambda vào SNS

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-northeast-1:466322313916:WebEnglishAlerts \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:ap-northeast-1:466322313916:function:ScaleWebEnglish

aws lambda add-permission \
  --function-name ScaleWebEnglish \
  --statement-id snsInvokePermission \
  --action "lambda:InvokeFunction" \
  --principal sns.amazonaws.com \
  --source-arn arn:aws:sns:ap-northeast-1:466322313916:WebEnglishAlerts
```

---

## ✅ 6. (Tùy chọn) Kích hoạt AWS DevOps Guru

```bash
aws devops-guru update-resource-collection \
  --action ADD \
  --resource-collection '{"CloudFormation":{"StackNames":["WebEnglishStack"]}}'
```

---

## ✅ 7. (Tùy chọn) Dùng AWS Step Functions

### State Machine ví dụ:

```json
{
  "Comment": "Scale ECS when high CPU",
  "StartAt": "ScaleService",
  "States": {
    "ScaleService": {
      "Type": "Task",
      "Resource": "arn:aws:states:::aws-sdk:ecs:updateService",
      "Parameters": {
        "Cluster": "MyCluster",
        "Service": "WebenglishService",
        "DesiredCount": 2
      },
      "End": true
    }
  }
}
```

---

## 🧪 8. Kiểm Thử

| Mô phỏng              | Công cụ                            |
| --------------------- | ---------------------------------- |
| Tăng CPU              | `stress-ng --cpu 4 --timeout 300s` |
| Kiểm tra log Lambda   | CloudWatch Logs → Lambda Logs      |
| Kiểm tra cảnh báo SNS | Email hoặc Logs SNS                |
| Xem ECS task count    | ECS Console hoặc CLI               |

---

## ✅ 9. Best Practices

| Thành phần | Khuyến nghị                      |
| ---------- | -------------------------------- |
| Alarm      | Kết hợp Anomaly Detection        |
| IAM        | Chỉ cấp quyền tối thiểu          |
| SNS        | Dùng đa channel (email + Lambda) |
| ECS        | Giới hạn scale lặp lại           |
