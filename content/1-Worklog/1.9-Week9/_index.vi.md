---
title: "Worklog Tuần 9"
date: 2026-07-13
weight: 9
chapter: false
pre: " <b> 1.9. </b> "
---

### Tuần 9: Lab — Serverless: Lambda & API Gateway

**Thời gian:** 06/07/2026 – 12/07/2026

#### Mục tiêu

* Dựng và công khai một endpoint HTTP thật mà không cần server
* Hiểu cold start đủ sâu để thiết kế né nó

#### Công việc đã thực hiện

* Viết Lambda function bằng Python, cấu hình biến môi trường, bộ nhớ và timeout, quan sát việc cấp phát bộ nhớ ảnh hưởng thế nào tới cả thời gian chạy lẫn chi phí
* Đo cold start so với lần gọi warm và đọc dòng `REPORT` trong CloudWatch Logs để thấy billed duration và max memory used
* Đặt API Gateway phía trước function, cấu hình resource, method, stage và CORS, gọi thử endpoint bằng `curl` và từ trình duyệt
* Viết execution role theo đặc quyền tối thiểu thay vì gắn một managed policy full-access

#### Kết quả đạt được

* **Output:** một endpoint kiểu `/moderate` đã deploy, trả về JSON, có log và metric hiện trong CloudWatch
* Bài lab này thành bản thiết kế trực tiếp cho backend của project ở mục 5.4
