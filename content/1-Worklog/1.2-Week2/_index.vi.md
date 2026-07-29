---
title: "Worklog Tuần 2"
date: 2026-07-13
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Tuần 2: Cày khoá học 2/3 — Compute, networking & storage

**Thời gian:** 18/05/2026 – 24/05/2026

#### Mục tiêu

* Học hết một phần ba thứ hai của khoá học
* Hiểu compute, network và storage ghép lại thành kiến trúc như thế nào

#### Công việc đã thực hiện

* Xem khoảng 100 video về EC2 (họ instance, AMI, EBS so với instance store, Elastic IP, key pair), Auto Scaling và Elastic Load Balancing
* VPC từ đầu tới cuối: quy hoạch CIDR, subnet public và private, route table, Internet Gateway, NAT Gateway, Security Group so với Network ACL
* Storage: các storage class và lifecycle rule của S3, bucket policy, versioning, EBS và EFS, và khi nào nên chọn cái nào
* Vẽ tay đường đi của một request trong ứng dụng web ba tầng để kiểm tra xem các khái niệm có khớp nhau không

#### Kết quả đạt được

* Đọc được sơ đồ kiến trúc AWS và giải thích được vì sao mỗi thành phần lại có mặt ở đó
* Hiểu tác động chi phí của NAT Gateway, EBS và lựa chọn storage class của S3 — kiến thức về sau giữ hoá đơn của project dưới 1 USD
