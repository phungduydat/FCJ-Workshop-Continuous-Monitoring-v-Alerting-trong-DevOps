---
title : "Dashboard Development"
date :  "`r Sys.Date()`" 
weight : 5 
chapter : false
pre : " <b> 5. </b> "
---

# 📊 Triển Khai Dashboard và Báo Cáo Trực Quan Cho Dự Án WebEnglish

Tài liệu này cung cấp quy trình chuyên nghiệp và chi tiết để thiết lập các bảng điều khiển (dashboards) và báo cáo giám sát dữ liệu hệ thống trong dự án WebEnglish. Việc giám sát được thực hiện bằng Amazon CloudWatch và Amazon QuickSight để cung cấp khả năng hiển thị thời gian thực cũng như các phân tích chiến lược giúp đội DevOps phản ứng nhanh chóng với sự cố sản phẩm.

---

## 🎯 Mục Tiêu

- Hiển thị hiệu suất hệ thống EC2, container, ứng dụng Spring Boot, và MySQL.
- Cảnh báo khi có dấu hiệu bất thường dựa trên hành vi (anomaly).
- Cung cấp báo cáo trực quan định kỳ cho DevOps và quản lý cấp cao.

---

## 🛠️ 1. Tạo Dashboard Trên Amazon CloudWatch

### ✅ Bước 1: Tạo Dashboard

1. Vào **AWS Console > CloudWatch > Dashboards**
2. Chọn **Create dashboard**
3. Đặt tên: `WebEnglish-Dashboard`
4. Chọn loại widget: `Line`, `Number`, `Text`, hoặc `Log query`

### ✅ Bước 2: Thêm Widget Giám Sát

#### 1. Widget CPU và Memory EC2

```json
{
  "type": "metric",
  "x": 0,
  "y": 0,
  "width": 12,
  "height": 6,
  "properties": {
    "metrics": [
      [ "WebEnglishMetrics", "cpu_usage_active" ],
      [ "WebEnglishMetrics", "mem_used_percent" ]
    ],
    "region": "ap-northeast-1",
    "title": "Hiệu suất Hệ Thống EC2",
    "view": "timeSeries",
    "stacked": false
  }
}
```

#### 2. Widget Logs Ứng Dụng

```json
{
  "type": "log",
  "x": 12,
  "y": 0,
  "width": 12,
  "height": 6,
  "properties": {
    "query": "SOURCE 'WebEnglishLogs' | filter @logStream like 'app'",
    "region": "ap-northeast-1",
    "title": "WebEnglish Application Logs"
  }
}
```

---

## 📈 2. Phân Tích Báo Cáo Qua Amazon QuickSight

### ✅ Bước 1: Xuất Logs Từ CloudWatch Về Amazon S3

1. Vào **CloudWatch > Log Groups**
2. Chọn nhóm log `WebEnglishLogs`
3. Bấm nút `Export data to Amazon S3`
4. Chọn S3 Bucket (đã tạo trước, ví dụ `webenglish-monitoring-bucket`)

### ✅ Bước 2: Tạo Dataset Trên QuickSight

1. Vào **Amazon QuickSight > Manage Data > New Dataset**
2. Chọn nguồn: `S3`
3. Nhập manifest.json hoặc đường dẫn trực tiếp tới bucket chứa dữ liệu CloudWatch logs
4. Tạo bảng dữ liệu `WebEnglishLogData`

### ✅ Bước 3: Tạo Báo Cáo

- Dùng biểu đồ cột: Lỗi phân theo ngày/giờ
- Dùng biểu đồ đường: Tốc độ truy vấn MySQL, độ trễ ứng dụng
- Lọc theo từ khóa lỗi: `"ERROR"`, `"Exception"`...

---

## 🧠 3. Chiến Lược Tối Ưu Hóa Cảnh Báo (Reduce False Positives/Negatives)

| Kỹ thuật                    | Mô tả |
|-----------------------------|-------|
| **Anomaly Detection**       | Sử dụng CloudWatch Anomaly Detector để tự học hành vi bình thường và phát hiện lệch chuẩn. |
| **Composite Alarm**         | Kết hợp nhiều chỉ số trong một cảnh báo để tránh cảnh báo sai khi chỉ 1 thành phần lỗi. |
| **Smoothing / Thống kê**    | Dùng trung bình, phần trăm (P90, P95) để làm mượt dữ liệu và loại bỏ spike ngắn. |
| **Delay đánh giá**          | Thiết lập `evaluationPeriods` lớn hơn (ví dụ 2–3 lần) để xác nhận tình trạng lỗi. |
| **Giới hạn lặp cảnh báo**   | Dùng `alarm suppression` để tránh spam nếu lỗi vẫn đang tiếp diễn. |

---

## 🧾 Mẹo Vận Hành

- ⏱️ Giới hạn thời gian giữ logs: `7–14 ngày` để giảm chi phí.
- 🔔 Cấu hình gửi cảnh báo qua **SNS Topic** tới email hoặc Lambda handler.
- 🧪 Kết hợp với **AWS DevOps Guru** để tự động phát hiện root cause.

---

## 📚 Tài Liệu Tham Khảo

- [CloudWatch Dashboards](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Dashboards.html)
- [Amazon QuickSight Getting Started](https://docs.aws.amazon.com/quicksight/latest/user/welcome.html)
- [Anomaly Detection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html)
