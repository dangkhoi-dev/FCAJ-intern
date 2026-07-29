---
title: "Worklog Tuần 6"
date: 2026-07-13
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Tuần 6: Lab — Networking & VPC

**Thời gian:** 15/06/2026 – 21/06/2026

#### Mục tiêu

* Dựng VPC từ một dải CIDR trống thay vì xài VPC mặc định
* Hiểu điều gì thực sự làm cho một subnet trở thành public

#### Công việc đã thực hiện

* Thiết kế và tạo VPC gồm subnet public và private trải trên hai Availability Zone
* Gắn Internet Gateway, viết route table, và xác nhận rằng một subnet là public chỉ nhờ route của nó — không phải nhờ một ô tick nào cả
* Đặt một instance vào subnet private rồi cho nó ra internet qua NAT Gateway, đo chi phí theo giờ của NAT Gateway đó rồi xoá đi
* So sánh Security Group (stateful) với Network ACL (stateless) bằng cách cố tình chặn chiều trả về

#### Kết quả đạt được

* **Output:** một VPC hai tầng chạy được, đã kiểm chứng cách ly public/private
* Hiểu mô hình mạng đủ chắc để bảo vệ quyết định cho Lambda chạy ngoài VPC trong project — vừa tránh chi phí NAT vừa tránh phạt cold start do ENI
