---
title: "Tự đánh giá"
date: 2026-07-13
weight: 6
chapter: false
pre: " <b> 6. </b> "
---

Trong thời gian thực tập tại chương trình **First Cloud AI Journey (AWS Việt Nam)** từ **15/06/2026** đến **14/08/2026**, tôi đã có cơ hội áp dụng kiến thức được học vào một dự án thực tế hoàn chỉnh — hệ thống kiểm duyệt ngôn từ tục tĩu serverless **Toxic Text Moderation Platform**.

### Vai trò của tôi trong nhóm

Tôi là **nhóm trưởng** của nhóm 4 thành viên. Cụ thể, tôi phụ trách:

**1. Huấn luyện model.** Tôi làm trọn phần model từ đầu tới cuối: chuẩn bị và làm sạch dữ liệu ViHSD, dựng baseline TF-IDF + Logistic Regression để đối chứng, fine-tune XLM-RoBERTa-base có class weighting cho tỉ lệ nhãn lệch 82/7/11, đánh giá trên tập test tách riêng, rồi xuất kết quả sang ONNX kèm quantize INT8 động để vừa được vào một Lambda container image.

**2. Định hướng kỹ thuật.** Tôi chọn bài toán và chọn stack, và ra những quyết định mà phần việc còn lại phụ thuộc vào: dùng XLM-RoBERTa thay vì PhoBERT (để có khả năng zero-shot tiếng Anh), tự host model đã quantize thay vì dùng inference endpoint quản trị sẵn (để tiết kiệm chi phí), và quan trọng nhất là quyết định giữ model fine-tune trong bản production dù nó thấp hơn baseline TF-IDF về macro-F1 — bởi recall của nó trên OFFENSIVE và HATE cao hơn đáng kể, và một hệ thống kiểm duyệt thì nên sai theo hướng bắt nhầm còn hơn bỏ sót.

**3. Tích hợp model lên chung với Bedrock.** Cơ chế cascade là do tôi thiết kế và tự triển khai. Thay vì coi precision thấp của model là một khuyết điểm phải giấu, tôi dựng kiến trúc xoay quanh chính điểm yếu đó: container Lambda chạy model ONNX cho mọi request, và chỉ những dự đoán dưới ngưỡng tin cậy 0,7 — đúng vùng model không đáng tin — mới được đẩy sang Claude Haiku trên Bedrock để thẩm định lại. Tôi viết phần logic ngưỡng, prompt gửi Bedrock và đoạn hợp nhất kết quả trả về, đồng thời tinh chỉnh ngưỡng dựa trên tập validation.

**4. Kiểm soát và test luồng của kiến trúc.** Tôi chịu trách nhiệm về tính đúng đắn của luồng end-to-end — Amplify → API Gateway → Lambda → (model | Bedrock) → DynamoDB → phản hồi. Tôi test luồng ở máy local bằng Lambda Runtime Interface Emulator trước khi deploy, viết ma trận test case phủ các đầu vào sạch, xúc phạm, thù ghét, tiếng Anh và các ca biên, kiểm chứng từng chặng trong CloudWatch Logs, và rà lại các IAM policy để giữ role của Lambda ở mức đặc quyền tối thiểu.

Song song đó tôi chấp bút và đăng các bài blog của nhóm lên cộng đồng AWS Study Group, chủ trì outline báo cáo, hoàn thiện và deploy website báo cáo song ngữ này.

Ba thành viên còn lại phụ trách các mảng ngoài phần model: Quân làm backend Lambda/API Gateway và nền tảng tài khoản AWS, Đức làm DynamoDB, IAM và ECR, Quốc làm front end React. Phần việc của tôi nằm ở chỗ giao nhau giữa tất cả các mảng đó — cũng chính là nơi phát sinh phần lớn lỗi tích hợp.

### Mức độ tham gia chương trình

Trong kỳ thực tập, chương trình mở **32 buổi** trên hệ thống đăng ký; tôi đã đăng ký **27 buổi**. Do tỉ lệ duyệt thấp so với nhu cầu, những buổi tôi thực sự được duyệt và có mặt trực tiếp tại văn phòng AWS Việt Nam là:

| Ngày | Loại buổi |
|---|---|
| 04/06/2026 | Buổi study |
| 05/06/2026 | Buổi study |
| 12/06/2026 | Buổi study |
| 29/06/2026 | Buổi study |
| 25/07/2026 | Sự kiện cộng đồng + 2 buổi study |

Tức là **7 buổi trải trên 5 ngày có mặt tại văn phòng**. Ngoài ra tôi còn tham dự các buổi cộng đồng ngày 06/06 và 27/06 được ghi lại ở mục 4. Tôi ghi kèm con số đăng ký bên cạnh con số tham dự vì khoảng chênh này phản ánh việc phân bổ chỗ chứ không phải mức độ tham gia của tôi — trong 32 buổi được mở, tôi đã ghi tên vào 27 buổi.

Để phản ánh khách quan quá trình thực tập, tôi tự đánh giá theo các tiêu chí sau:

| STT | Tiêu chí | Mô tả | Tốt | Khá | Trung bình |
| --- | --- | --- | --- | --- | --- |
| 1 | **Kiến thức và kỹ năng chuyên môn** | Fine-tune model NLP, tối ưu ONNX, đóng gói serverless, sử dụng thành thạo các dịch vụ AWS của dự án | ✅ | ☐ | ☐ |
| 2 | **Khả năng học hỏi** | Tiếp thu nhanh các công nghệ mới (Bedrock, Lambda container, quantization) trong thời gian ngắn | ✅ | ☐ | ☐ |
| 3 | **Chủ động** | Tự nghiên cứu giải pháp, đề xuất chuyển từ PhoBERT sang XLM-R, chủ động nhận phần việc khó | ✅ | ☐ | ☐ |
| 4 | **Tinh thần trách nhiệm** | Hoàn thành các task được phân công đúng deadline của nhóm | ✅ | ☐ | ☐ |
| 5 | **Kỷ luật** | Tuân thủ giờ giấc và quy định của chương trình | ☐ | ✅ | ☐ |
| 6 | **Tính cầu tiến** | Lắng nghe feedback từ mentor và các thành viên để điều chỉnh | ☐ | ✅ | ☐ |
| 7 | **Giao tiếp** | Trình bày kỹ thuật rõ ràng qua báo cáo, blog và trao đổi nhóm | ☐ | ✅ | ☐ |
| 8 | **Hợp tác nhóm** | Phối hợp chặt với Quốc (UI), Quân và Đức (backend/hạ tầng) ở các điểm tích hợp | ✅ | ☐ | ☐ |

Điểm tôi thấy mình cần cải thiện nhất là quản lý thời gian ở giai đoạn cuối dự án — một số hạng mục báo cáo dồn sát deadline. Nếu được làm lại, tôi sẽ viết tài liệu song song với quá trình triển khai thay vì để dồn về cuối.
