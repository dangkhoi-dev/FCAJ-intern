---
title: "Giới thiệu"
date: 2026-07-13
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

#### Bài toán

Ngôn từ tục tĩu, xúc phạm xuất hiện dày đặc trong bình luận trên các nền tảng mạng xã hội Việt Nam. Kiểm duyệt thủ công không theo kịp tốc độ nội dung, còn các API thương mại nhóm đã khảo sát đều tối ưu cho tiếng Anh là chính, nên teencode và biến thể ký tự tiếng Việt (ví dụ: "đmm", "vkl", "cl") là nhóm ca chúng xử lý kém ổn định nhất. Workshop này giải quyết bài toán đó bằng một hệ thống kiểm duyệt tự động, song ngữ Việt–Anh, chi phí gần bằng 0.

#### Kiến trúc tổng thể

Luồng xử lý của hệ thống:

1. Người dùng nhập câu trên **website demo** (React, host bằng Amplify Hosting).
2. **API Gateway** nhận request `POST /moderate` và chuyển cho Lambda.
3. **Lambda (container image)** chạy mô hình XLM-RoBERTa đã fine-tune (ONNX INT8), trả về nhãn và confidence.
4. Nếu confidence < 0.7, Lambda gọi **Amazon Bedrock (Claude Haiku)** để phân xử lại.
5. Kết quả được ghi vào **DynamoDB** và trả về UI, từ vi phạm được highlight.
6. **CloudWatch** ghi log, metric và cảnh báo; **S3** lưu dataset và model artifacts; **ECR** chứa Docker image.

![Sơ đồ kiến trúc](/images/5-Workshop/architecture.png)

#### Các dịch vụ AWS sử dụng và lý do lựa chọn

| Dịch vụ | Vai trò | Lý do chọn |
|---|---|---|
| Amplify Hosting | Host UI demo | CI/CD từ GitHub, HTTPS sẵn, miễn phí mức thấp |
| API Gateway | REST endpoint | Managed, throttling chống lạm dụng |
| Lambda (container) | Suy luận model | Serverless, trả tiền theo request, hỗ trợ image tới 10 GB |
| Amazon Bedrock | Phân xử câu khó | Dùng Claude không cần tự host LLM |
| DynamoDB | Lưu lịch sử kiểm duyệt | On-demand, free tier, độ trễ ms |
| S3 | Dataset + model artifacts | Bền vững, rẻ |
| ECR | Chứa Docker image | Tích hợp trực tiếp với Lambda |
| CloudWatch | Log / metric / alarm | Giám sát và đo lường theo yêu cầu project |
| Organizations + IAM Identity Center | Quản lý 4 account thành viên | Least privilege, tách môi trường |

#### Kết quả đạt được sau workshop

+ Website demo public: nhập câu → nhận kết quả phân loại kèm highlight trong ~1 giây.
+ API `/moderate` có thể tích hợp vào ứng dụng khác.
+ Hiểu cách kết hợp mô hình ML tự train với LLM trên Bedrock trong một pipeline serverless hoàn chỉnh.
