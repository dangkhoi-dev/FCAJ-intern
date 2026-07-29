---
title: "Bản đề xuất"
date: 2026-07-13
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Toxic Text Moderation Platform
## Hệ thống nhận diện ngôn từ tục tĩu / độc hại serverless trên AWS

### 1. Tóm tắt điều hành
Toxic Text Moderation Platform là hệ thống kiểm duyệt nội dung văn bản thời gian thực do nhóm 4 thành viên (Đức, Quốc, Khôi, Quân) xây dựng. Hệ thống nhận diện ngôn từ tục tĩu, xúc phạm trong bình luận **tiếng Việt và tiếng Anh** bằng mô hình text classifier do nhóm tự huấn luyện, kết hợp Amazon Bedrock làm "trọng tài" cho các trường hợp mô hình không chắc chắn. Toàn bộ hạ tầng chạy serverless (Lambda, API Gateway, DynamoDB, Amplify), chi phí gần như bằng 0 khi không có traffic, và người dùng có thể trải nghiệm trực tiếp qua website demo.

### 2. Tuyên bố vấn đề
*Vấn đề hiện tại*
Các nền tảng có nội dung do người dùng tạo (bình luận, chat, diễn đàn) tại Việt Nam phải kiểm duyệt thủ công lượng lớn ngôn từ tục tĩu, xúc phạm. Kiểm duyệt thủ công chậm, tốn nhân lực và không nhất quán; còn các API kiểm duyệt thương mại nhóm đã khảo sát đều tối ưu cho tiếng Anh là chính, nên teencode, viết tắt và biến thể ký tự tiếng Việt như "đmm", "vcl" là nhóm ca chúng xử lý kém ổn định nhất.

*Giải pháp*
Nhóm tự fine-tune mô hình phân loại văn bản đa ngôn ngữ trên dataset công khai tiếng Việt **ViHSD**, đóng gói vào Lambda container để suy luận nhanh với chi phí thấp. Khi độ tin cậy (confidence) của mô hình thấp hơn ngưỡng, hệ thống chuyển câu sang Amazon Bedrock (Claude) để phân xử — cân bằng giữa tốc độ, chi phí và độ chính xác. Kết quả được lưu vào DynamoDB để thống kê và hiển thị lịch sử trên UI.

*Lợi ích*
Giảm khối lượng kiểm duyệt thủ công, trả kết quả trong khoảng 1 giây khi container đã warm, hỗ trợ tiếng Việt và — nhờ backbone đa ngôn ngữ — cả tiếng Anh, kiến trúc serverless không tốn chi phí cố định và dễ mở rộng thành API kiểm duyệt cho ứng dụng khác.

### 3. Kiến trúc giải pháp
Luồng xử lý: người dùng nhập câu trên website (Amplify Hosting) → API Gateway nhận request POST /moderate → Lambda (container image chứa mô hình đã fine-tune) phân loại → nếu confidence < 0.7, Lambda gọi Amazon Bedrock (Claude Haiku) phân xử lại → kết quả (nhãn, confidence, nguồn quyết định) ghi vào DynamoDB và trả về UI, các từ tục tĩu được highlight.

![Sơ đồ kiến trúc](/images/2-Proposal/architecture.png)
*(Sơ đồ do Đức & Quân vẽ trên draw.io.)*

*Dịch vụ AWS sử dụng*
- **AWS Amplify Hosting**: host UI React demo, CI/CD từ GitHub, HTTPS sẵn có.
- **Amazon API Gateway**: REST API cho endpoint /moderate, throttling chống lạm dụng.
- **AWS Lambda (container image)**: chạy suy luận mô hình XLM-RoBERTa đã fine-tune (ONNX INT8); serverless, chỉ trả tiền theo request.
- **Amazon Bedrock (Claude Haiku)**: LLM phân xử các câu khó (mỉa mai, tiếng lóng mới), không cần tự host LLM.
- **Amazon DynamoDB**: lưu lịch sử kiểm duyệt (requestId, câu, nhãn, confidence, timestamp).
- **Amazon S3**: lưu dataset và model artifacts.
- **Amazon ECR**: chứa Docker image của Lambda.
- **Amazon CloudWatch**: logs, metrics, alarm (lỗi Lambda, độ trễ, chi phí).
- **AWS Organizations + IAM Identity Center**: quản lý tài khoản 4 thành viên, phân quyền least privilege.

*Lý do chọn kiến trúc serverless*: không phải quản lý server, tự động scale theo traffic, chi phí theo mức dùng phù hợp dự án sinh viên, và thể hiện được nhiều dịch vụ AWS trong một use-case thực tế (vượt yêu cầu tối thiểu 3 dịch vụ).

### 4. Mô hình AI và dữ liệu
- **Dataset**: **ViHSD** (~33.000 bình luận tiếng Việt, nhãn CLEAN/OFFENSIVE/HATE) là dataset nhóm thực sự dùng để huấn luyện, giữ nguyên cách chia train/val/test gốc của dataset. ViCTSD và Jigsaw Toxic Comment Classification có được khảo sát như hướng mở rộng nhưng không dùng trong lần train cuối — backbone đa ngôn ngữ đã xử lý được tiếng Anh theo kiểu zero-shot.
- **Baseline**: TF-IDF + Logistic Regression để có mốc so sánh.
- **Mô hình chính**: fine-tune **XLM-RoBERTa-base**. Ban đầu nhóm đề xuất PhoBERT (vi) + DistilBERT (en), nhưng quyết định chuyển sang XLM-R vì một model duy nhất xử lý được đa ngôn ngữ (zero-shot cross-lingual) và đơn giản hóa triển khai; xuất ONNX/quantize INT8 để giảm cold-start trên Lambda.
- **Đánh giá**: Accuracy, Precision/Recall, F1-score, Confusion Matrix; mục tiêu F1 ≥ 0.85 trên tập test.

### 5. Timeline (15/07 – 31/07)
- **15–19/07 (Phase kỹ thuật)**: setup AWS Organization và budget alarm (Quân, Đức); thu thập và huấn luyện mô hình (Khôi, Quốc); dựng hạ tầng và backend (Đức, Quân); UI demo và deploy Amplify (Quốc, Khôi); test end-to-end.
- **20–31/07 (Phase báo cáo)**: viết báo cáo song ngữ theo template, vẽ biểu đồ kết quả, viết và đăng bài blog lên AWS Study Group, review chéo, clean-up tài nguyên.

### 6. Ước tính ngân sách
- AWS Lambda: ~0 USD (trong free tier, ~5.000 request demo).
- API Gateway: ~0,02 USD.
- DynamoDB (on-demand): ~0 USD (free tier).
- S3: ~0,05 USD (2–3 GB dataset + model).
- ECR: ~0,10 USD (1 image ~2 GB).
- Amplify Hosting: ~0,15 USD.
- Amazon Bedrock (Claude Haiku): ~0,50 USD — chỉ những câu model không chắc chắn (confidence < 0.7) mới gọi Bedrock, chiếm phần nhỏ tổng traffic, nên mức này đã tính dư an toàn.
- **Tổng ước tính: < 1 USD cho toàn bộ giai đoạn demo.** Huấn luyện mô hình thực hiện trên Google Colab / SageMaker Studio Lab (miễn phí) nên không phát sinh chi phí GPU.

### 7. Đánh giá rủi ro
- **Cold-start Lambda với image lớn** (ảnh hưởng trung bình, xác suất cao): quantize model, tăng memory Lambda, cân nhắc provisioned concurrency khi demo.
- **Model kém với tiếng lóng/teencode mới** (trung bình, trung bình): lớp Bedrock phân xử + bổ sung từ điển chuẩn hóa trước khi suy luận.
- **Vượt ngân sách Bedrock** (thấp, thấp): đặt ngưỡng confidence hợp lý, budget alarm, giới hạn độ dài input.
- **Trễ tiến độ** (trung bình, trung bình): phân công song song 2 nhánh tech (AWS vs AI/UI), có ngày 19/07 làm buffer.

### 8. Kết quả kỳ vọng
Website demo public cho phép nhập câu bất kỳ và nhận kết quả phân loại kèm highlight từ tục tĩu trong khoảng 1 giây khi container Lambda đã warm; báo cáo song ngữ đầy đủ theo template FCJ; 1 bài blog kỹ thuật chia sẻ trên AWS Study Group; toàn bộ hạ tầng có thể tái tạo theo hướng dẫn step-by-step trong mục Workshop.
