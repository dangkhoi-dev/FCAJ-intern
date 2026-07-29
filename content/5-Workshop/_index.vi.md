---
title: "Workshop"
date: 2026-07-13
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Xây dựng hệ thống nhận diện ngôn từ tục tĩu serverless với Lambda, Bedrock và DynamoDB

#### Tổng quan

Trong workshop này, chúng ta sẽ xây dựng một **hệ thống kiểm duyệt văn bản thời gian thực** hoàn chỉnh trên AWS: người dùng nhập một câu bất kỳ (tiếng Việt hoặc tiếng Anh) trên website, hệ thống phân loại câu đó là **CLEAN / OFFENSIVE / HATE** (bình thường / tục tĩu / thù ghét) và highlight các từ vi phạm.

Điểm đặc biệt của kiến trúc là cơ chế **hai tầng**:
+ **Tầng 1 — Model tự huấn luyện**: mô hình XLM-RoBERTa được nhóm fine-tune trên dataset công khai ViHSD (hate speech tiếng Việt), xuất ONNX INT8 và đóng gói thành Lambda container image — suy luận nhanh, gần như miễn phí trên CPU.
+ **Tầng 2 — Amazon Bedrock**: khi model không chắc chắn (confidence < 0.7), câu được chuyển cho Claude Haiku trên Bedrock phân xử — xử lý tốt các trường hợp khó như mỉa mai, tiếng lóng mới, teencode.

Kết quả kiểm duyệt được lưu vào **DynamoDB** và hiển thị lịch sử trên UI React host bằng **Amplify Hosting**.

![Sơ đồ kiến trúc](/images/5-Workshop/architecture.png)

#### Nội dung

1. [Tổng quan về workshop](5.1-workshop-overview/)
2. [Chuẩn bị](5.2-prerequiste/)
3. [Huấn luyện và đóng gói model](5.3-train-package-model/)
4. [Triển khai backend: DynamoDB, Lambda, API Gateway, Bedrock](5.4-deploy-backend/)
5. [Deploy frontend & kiểm thử end-to-end](5.5-deploy-frontend-test/)
6. [Dọn dẹp tài nguyên](5.6-cleanup/)
