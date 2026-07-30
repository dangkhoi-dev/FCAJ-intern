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

1. Trình duyệt tải ứng dụng React từ **Amplify Hosting** — chỉ là file tĩnh.
2. Chính trình duyệt gửi `POST /moderate` **thẳng tới API Gateway**, ở origin khác với trang vừa tải. Amplify không phải proxy và từ đây không còn nằm trên đường đi; chính lời gọi cross-origin này là lý do phải cấu hình CORS.
3. **API Gateway** gọi **Lambda (container image)** qua proxy integration.
4. Lambda chạy model XLM-RoBERTa đã fine-tune (ONNX INT8) nằm sẵn trong image, cho ra nhãn kèm độ tin cậy.
5. Nếu độ tin cậy < 0.7, request được đẩy sang **Amazon Bedrock (Claude 3 Haiku)**.
6. Bedrock trả nhãn đã phân xử về lại Lambda.
7. Dù đi nhánh nào, Lambda cũng ghi bản ghi vào **DynamoDB** đúng một lần.
8. Kết quả trả ngược qua API Gateway về trình duyệt, các từ vi phạm được highlight.

Song song với luồng request: **IAM** cấp execution role theo đặc quyền tối thiểu, **CloudWatch** thu log, metric và cảnh báo, **CloudTrail** ghi nhật ký hoạt động API ở cấp tài khoản. Chỉ ở giai đoạn build, **S3** phân phối model artifact và **ECR** chứa container image — Lambda pull và cache image lúc tạo/cập nhật function, không phải mỗi request.

![Sơ đồ kiến trúc](/images/5-Workshop/architecture.png)

#### Các dịch vụ AWS sử dụng và lý do lựa chọn

| Dịch vụ | Vai trò | Lý do chọn |
|---|---|---|
| Amplify Hosting | Host UI demo | CI/CD từ GitHub, HTTPS sẵn, miễn phí mức thấp |
| API Gateway | REST endpoint | Managed, throttling chống lạm dụng |
| Lambda (container) | Suy luận model | Serverless, trả tiền theo request, hỗ trợ image tới 10 GB |
| Amazon Bedrock | Phân xử câu khó | Dùng Claude không cần tự host LLM |
| DynamoDB | Lưu lịch sử kiểm duyệt | On-demand, free tier, độ trễ ms |
| S3 | Dataset + model artifact (**chỉ ở giai đoạn build**) | Bền, rẻ; trọng số nằm sẵn trong image, không tải lúc chạy |
| ECR | Chứa container image | Tích hợp trực tiếp với Lambda; image được cache lúc tạo/cập nhật function, không phải mỗi request |
| CloudWatch | Log / metric / alarm | Giám sát và đo lường theo yêu cầu project |
| CloudTrail | Nhật ký kiểm toán API cấp tài khoản | Mọi thao tác truy được về một danh tính cụ thể |
| Organizations + IAM Identity Center | Quản lý 4 account thành viên | Least privilege, tách môi trường |

#### Kết quả đạt được sau workshop

+ Website demo public: nhập câu → nhận kết quả phân loại kèm highlight trong ~1 giây.
+ API `/moderate` có thể tích hợp vào ứng dụng khác.
+ Hiểu cách kết hợp mô hình ML tự train với LLM trên Bedrock trong một pipeline serverless hoàn chỉnh.
