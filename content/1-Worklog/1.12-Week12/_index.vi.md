---
title: "Worklog Tuần 12"
date: 2026-07-13
weight: 12
chapter: false
pre: " <b> 1.12. </b> "
---

### Tuần 12: Thực thi project — xây dựng, kiểm thử, hoàn thiện & nộp

**Thời gian:** 27/07/2026 – 31/07/2026

#### Mục tiêu

* Đưa Toxic Text Moderation Platform chạy thông từ đầu tới cuối
* Hoàn thiện báo cáo song ngữ, đăng blog, dọn tài nguyên và nộp bài

#### Công việc đã thực hiện

* Fine-tune model XLM-RoBERTa bản cuối, export sang ONNX và quantize INT8 động để vừa container Lambda
* Deploy backend: Lambda container image + API Gateway + DynamoDB, kèm cơ chế cascade sang Amazon Bedrock theo ngưỡng độ tin cậy
* Dựng front end React và deploy lên Amplify Hosting; kiểm thử end-to-end và các ca lỗi với cả đầu vào tiếng Việt lẫn tiếng Anh
* Bật CloudWatch log, metric và alarm lỗi; chụp trọn bộ ảnh minh chứng
* Viết báo cáo song ngữ này trên template FCJ, đăng bài blog của nhóm lên cộng đồng AWS Study Group
* Review chéo trong nhóm, deploy trang báo cáo lên GitHub Pages, sao lưu mã nguồn và notebook lên GitHub
* Xoá toàn bộ tài nguyên AWS, kiểm tra con số Billing cuối cùng và nộp link qua FCAJ Portal ngày 31/07

#### Kết quả đạt được

* **Output:** demo công khai chạy thông từ đầu tới cuối kèm giám sát đầy đủ, báo cáo song ngữ hoàn chỉnh, bài blog đã đăng, tài nguyên đã dọn sạch với chi phí gần bằng 0
* Kết quả đo được báo cáo trung thực, kể cả chỗ model chưa đạt mục tiêu của proposal và lý do vì sao
