---
title: "Worklog Tuần 8"
date: 2026-07-13
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

### Tuần 8: Lab — Database: RDS & DynamoDB

**Thời gian:** 29/06/2026 – 05/07/2026

#### Mục tiêu

* So sánh một CSDL quan hệ và một CSDL NoSQL quản trị sẵn trên cùng một workload
* Thiết kế bảng lưu lịch sử kiểm duyệt cho project

#### Công việc đã thực hiện

* Khởi tạo một RDS MySQL, kết nối từ EC2, nạp một bộ dữ liệu nhỏ, rồi thử khôi phục từ snapshot và đo chi phí khi để nó chạy không
* Tạo bảng DynamoDB và thử nghiệm thiết kế partition key; cố tình tạo hot partition để xem throughput bị ảnh hưởng ra sao
* Tạo Global Secondary Index để phục vụ một truy vấn mà bảng gốc không đáp ứng được
* So sánh billing provisioned với on-demand trên một mẫu truy cập giật cục, khó đoán

#### Kết quả đạt được

* **Output:** thiết kế bảng `ModerationHistory` — partition key `requestId`, GSI `timestamp-index`, on-demand billing — đưa thẳng vào project không phải sửa
* Kết luận: với workload demo giật cục, DynamoDB on-demand gần như không tốn tiền, còn một RDS instance để không thì có
