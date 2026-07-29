---
title: "Worklog Tuần 5"
date: 2026-07-13
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Tuần 5: Lab — Compute & EC2

**Thời gian:** 08/06/2026 – 14/06/2026

#### Mục tiêu

* Thực hành trọn vòng đời một EC2 instance từ console và từ CLI
* Đo xem Free Tier thực sự phủ tới đâu

#### Công việc đã thực hiện

* Khởi tạo instance t3.micro từ cả console lẫn CLI, so sánh các AMI, kết nối bằng SSH và bằng Session Manager
* Gắn, format, mount, snapshot rồi khôi phục một EBS volume; quan sát cái gì sống sót qua stop/start và cái gì không
* Cấu hình Security Group chỉ mở cổng 22 cho đúng một IP, rồi chỉ mở cổng 80 cho mọi người, và kiểm chứng khác biệt bằng `curl`
* Dựng một Auto Scaling Group nhỏ sau Application Load Balancer, đo chi phí xong thì xoá

#### Kết quả đạt được

* **Output:** một web server chạy được qua ALB, kèm quy trình dọn tài nguyên đã ghi lại
* Hiểu trực tiếp vì sao về sau project chọn Lambda thay vì EC2: không tốn tiền lúc rảnh, không phải vá hệ điều hành, không phải tính dung lượng
