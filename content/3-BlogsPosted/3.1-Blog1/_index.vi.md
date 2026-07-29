---
title: "Blog 1"
date: 2026-07-13
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# Xây dựng hệ thống kiểm duyệt nội dung serverless với AWS Lambda và Amazon Bedrock

**Người viết & đăng bài:** Trần Phan Đăng Khôi &emsp;|&emsp; **Nơi đăng:** Group Facebook AWS Study Group

**Link bài đăng:** đăng ngày 28/07/2026 trong group [AWS Study Group VN](https://www.facebook.com/groups/awsstudygroupfcj) — đang chờ quản trị viên duyệt tại thời điểm nộp báo cáo

#### Tóm tắt nội dung

Bài viết giới thiệu kiến trúc tổng thể của hệ thống FCAJ: luồng xử lý từ React UI (Amplify) qua API Gateway đến Lambda container chạy model phân loại, cơ chế "trọng tài" gọi Bedrock Claude Haiku khi model không chắc chắn (confidence < 0.7), và cách ghi lịch sử vào DynamoDB. Bài viết nhấn mạnh pattern cascade — model nhỏ xử lý phần lớn request, LLM chỉ xử lý ca khó — giúp cân bằng chi phí và độ chính xác, cùng các bài học thực tế: Lambda container image cho model AI, cold start, request Bedrock model access và thiết kế IAM least privilege.

#### Ảnh bài đăng

![Blog 1 trên AWS Study Group](/images/3-BlogsPosted/blog1.png)
