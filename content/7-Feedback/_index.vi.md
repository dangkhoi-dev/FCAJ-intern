---
title: "Chia sẻ, đóng góp ý kiến"
date: 2026-07-13
weight: 7
chapter: false
pre: " <b> 7. </b> "
---

Dưới đây là những chia sẻ và góp ý của tôi sau quá trình tham gia chương trình First Cloud AI Journey, hy vọng giúp team FCAJ hoàn thiện chương trình hơn cho các khóa sau.

**1. Môi trường học tập và làm việc**
Môi trường của chương trình rất cởi mở và khuyến khích tự học. Điều tôi đánh giá cao nhất là cách chương trình để nhóm tự chịu trách nhiệm end-to-end với project của mình — từ ý tưởng, kiến trúc đến triển khai và chi phí — giống một dự án thật thay vì một bài tập.

**2. Sự hỗ trợ của mentor / team admin**
Mentor phản hồi nhanh và định hướng đúng lúc, đặc biệt ở giai đoạn chốt kiến trúc. Thay vì đưa đáp án, mentor thường đặt câu hỏi ngược để nhóm tự tìm ra vấn đề — cách hướng dẫn này ban đầu hơi "khó chịu" nhưng nhìn lại là điều giúp tôi tiến bộ nhanh nhất.

**3. Sự phù hợp giữa nội dung và định hướng nghề nghiệp**
Lộ trình từ dịch vụ AWS cơ bản đến project AI serverless rất khớp với định hướng của tôi. Việc bắt buộc dùng nhiều dịch vụ trong một use-case thực tế giúp kiến thức không rời rạc mà kết nối thành hệ thống.

**4. Cơ hội học hỏi và phát triển kỹ năng**
Ngoài kỹ thuật, tôi học được nhiều kỹ năng mềm: phân công và phối hợp trong nhóm 4 người qua bảng công việc chung, viết tài liệu song ngữ, viết blog kỹ thuật cho cộng đồng, và quản lý chi phí cloud có kỷ luật.

**5. Góp ý cải thiện**
Tôi mong chương trình bổ sung: (1) một buổi hướng dẫn sớm về Amazon Bedrock và các giới hạn/model access theo region — nhóm mất kha khá thời gian tự mò phần này; (2) quota GPU hoặc hướng dẫn SageMaker Studio Lab chi tiết hơn cho các nhóm làm đề tài AI; (3) template báo cáo cập nhật sẵn tương thích với phiên bản Hugo mới để đỡ gặp lỗi build khi làm website báo cáo.

**7. Sau khi nộp — giai đoạn tiếp nhận phản hồi và phát triển thêm (01/08 – 14/08/2026)**
Báo cáo được nộp ngày 31/07/2026, nhưng kỳ Thực tập ngoài trường của Trường kéo dài tới 14/08/2026. Nhóm dùng hai tuần còn lại để khép vòng lặp thay vì dừng lại:

* **Thu thập phản hồi.** Nhóm tổng hợp bình luận về ba bài blog từ cộng đồng AWS Study Group và các câu hỏi nhận được trong buổi cộng đồng ngày 25/07, rồi biến chúng thành một danh sách vấn đề đã sắp thứ tự ưu tiên.
* **Xử lý kết quả model.** Phản hồi rõ nhất nằm ở phần đánh giá của mục 5.3 — model fine-tune thấp hơn baseline TF-IDF về macro-F1 vì Colab bản miễn phí chặn việc train ở 3 epoch. Trong giai đoạn này nhóm train lại trên runtime GPU trả phí kèm early stopping theo macro-F1, và bắt đầu đối chứng với một encoder pretrain riêng cho tiếng Việt. Các kết quả đó không được đưa vào báo cáo chấm điểm, vì báo cáo chỉ trình bày những gì đã đo được tính tới ngày nộp.
* **Làm chắc hệ thống.** Giảm cold start bằng cách tinh gọn container image, thêm kiểm tra độ dài đầu vào ở phía server thay vì chỉ dựa vào `maxLength` của trình duyệt, và thêm giới hạn tần suất theo IP ở API Gateway.
* **Tài liệu.** Viết runbook triển khai để một thành viên mới có thể dựng lại toàn bộ hệ thống từ một tài khoản AWS trống.

Giai đoạn này chính là mục đích của hai tuần sau ngày nộp, và cũng là nơi phần lớn các mục "nếu làm lại nhóm sẽ làm gì" ở mục 5.3 thực sự được thử nghiệm.

**6. Giới thiệu chương trình**
Tôi chắc chắn sẽ giới thiệu First Cloud AI Journey cho bạn bè cùng ngành — đây là một trong số ít chương trình cho sinh viên trải nghiệm trọn vẹn chu trình xây dựng sản phẩm cloud thực tế với chi phí gần như bằng 0.
