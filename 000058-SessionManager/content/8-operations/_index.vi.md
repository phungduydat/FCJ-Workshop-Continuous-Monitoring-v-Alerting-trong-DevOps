---
title : "Operational Procedures"
date :  "`r Sys.Date()`" 
weight : 8
chapter : false
pre : " <b> 8. </b> "
---

# ✅ Quy trình Vận hành & Tối ưu hóa (Operational Procedures & Optimization)

## 🔹 1. Thiết lập Runbook chuyên sâu

**Runbook** là tài liệu hướng dẫn chi tiết từng bước xử lý cho các kịch bản lỗi phổ biến. Tài liệu nên bao gồm:

| Kịch bản sự cố | Dấu hiệu nhận biết | Nguyên nhân phổ biến | Bước xử lý chi tiết | Người chịu trách nhiệm |
|---------------|--------------------|-----------------------|---------------------|--------------------------|
| CPU sử dụng 100% | Alarm CloudWatch “HighCPUUsage” | Do load đột biến từ người dùng | 1. Xác minh log ứng dụng <br> 2. Kiểm tra container ECS đang chạy <br> 3. Scale ECS service lên thêm 1 task | DevOps Engineer |
| Mất kết nối DB | Ứng dụng trả về lỗi `500` <br> Alarm về `MySQLConnectionTimeout` | DB quá tải hoặc crash container | 1. Kiểm tra container MySQL <br> 2. Khởi động lại container <br> 3. Kiểm tra lại trạng thái từ dashboard | Backend Developer |

> 📁 **Lưu trữ đề xuất**: `/runbook/webenglish/mysql-restart.md`

---

## 🔹 2. Phân tích log với CloudWatch Log Insights

### Câu lệnh truy vấn phổ biến

**Truy tìm lỗi ứng dụng**
```sql
fields @timestamp, @message
| filter @message like /Exception/ or /ERROR/
| sort @timestamp desc
| limit 20
```

**Kiểm tra lỗi kết nối DB**
```sql
fields @timestamp, @message
| filter @message like /Connection refused/ or /JDBC/
| sort @timestamp desc
| limit 50
```

> 📌 Lưu các truy vấn mẫu để DevOps dễ tra cứu và chia sẻ nội bộ.

---

## 🔹 3. Triển khai Auto Scaling Service

### ECS Fargate – Ví dụ cấu hình trong AWS CDK

```ts
service.autoScaleTaskCount({ maxCapacity: 5, minCapacity: 1 })
  .scaleOnCpuUtilization('CpuScaling', {
    targetUtilizationPercent: 60
  });
```

### CloudWatch Alarm để kích hoạt auto scaling

- CPU Utilization ≥ 75% → scale-out
- CPU Utilization ≤ 30% → scale-in

---

# ✅ Kiểm thử & Tối ưu hóa hệ thống

## 🔹 1. Mô phỏng kịch bản lỗi

### CPU Spike

```bash
sudo yum install stress -y
stress --cpu 2 --timeout 300
```

### Tạo lỗi ứng dụng

```bash
for i in {1..100}; do curl -s http://localhost:8080/ & done
```

### Mô phỏng crash MySQL

```bash
docker exec -it mysql_container pkill -9 mysqld
```

## 🔹 2. Quan sát và xác minh hệ thống

- Cảnh báo CloudWatch có được kích hoạt?
- SNS có gửi email đến đúng đối tượng?
- Lambda hoặc Auto Scaling có phản hồi?
- Log có ghi đầy đủ lỗi?

## 🔹 3. Tinh chỉnh ngưỡng cảnh báo

| Metric | Threshold ban đầu | Điều chỉnh đề xuất |
|--------|-------------------|--------------------|
| cpu_usage_active | > 80% | > 85% nếu có spike thường xuyên |
| memory usage | > 75% | > 70% nếu ứng dụng Spring Boot memory-intensive |
| MySQLQueries | > 1000/min | Điều chỉnh dựa trên traffic thực tế |

---

## 📈 Lợi ích & Báo cáo KPIs

| Chỉ Số | Mục Tiêu |
|--------|----------|
| Thời gian phản hồi sự cố | < 10 phút |
| % lỗi được phát hiện tự động | > 95% |
| Thời gian xử lý log | < 3 phút |
| False positive rate | < 5% |

---

## 📌 Gợi ý mở rộng

- Tích hợp AWS DevOps Guru để phát hiện nguyên nhân lỗi (root cause).
- Tự động gửi báo cáo định kỳ qua email bằng Amazon QuickSight + SNS.
- Tự động xử lý lỗi bằng AWS Lambda hoặc AWS Systems Manager Run Command.

---

## 📚 Tài liệu tham khảo

- [AWS CloudWatch Log Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)
- [Auto Scaling ECS](https://docs.aws.amazon.com/autoscaling/application/userguide/application-auto-scaling.html)
- [AWS Runbook Template](https://aws.amazon.com/blogs/devops/using-aws-systems-manager-to-create-runbooks/)
