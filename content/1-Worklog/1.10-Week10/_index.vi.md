---
title: "Worklog Tuần 10"
date: 2026-07-13
weight: 10
chapter: false
pre: " <b> 1.10. </b> "
---

### Tuần 10: Lab — Container (Docker & ECR) và AI trên AWS

**Thời gian:** 13/07/2026 – 19/07/2026

#### Mục tiêu

* Đóng gói ứng dụng thành container image và chạy trên Lambda
* Đánh giá Bedrock và SageMaker so với yêu cầu của project

#### Công việc đã thực hiện

* Học vòng build/tag/push của Docker, viết Dockerfile multi-stage và push image lên private repository trên ECR
* Deploy Lambda function từ container image, test trước ở máy local bằng AWS Lambda Runtime Interface Emulator (RIE)
* Khám phá SageMaker notebook và endpoint, tính giá một inference endpoint chạy thường trực so với Lambda tính theo request
* Gọi Amazon Bedrock (Claude Haiku) từ Python, thử nghiệm thiết kế prompt cho bài toán phân loại, đo độ trễ và chi phí mỗi lần gọi

#### Kết quả đạt được

* **Output:** một container image trên ECR chạy thành công dưới dạng Lambda function, đã kiểm chứng ở local qua RIE
* Chốt kiến trúc: model tự host đã quantize nằm trong container Lambda lo đường đi thông thường, chỉ đẩy sang Bedrock khi độ tin cậy thấp — rẻ ở mặc định, chính xác khi cần
