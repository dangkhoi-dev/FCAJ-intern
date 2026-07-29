---
title: "Worklog Tuần 11"
date: 2026-07-13
weight: 11
chapter: false
pre: " <b> 1.11. </b> "
---

### Tuần 11: Fine-tune NLP, lên ý tưởng & viết proposal

**Thời gian:** 20/07/2026 – 26/07/2026

#### Mục tiêu

* Chọn bài toán cho project cuối khoá và chứng minh hướng model là khả thi
* Được duyệt proposal

#### Công việc đã thực hiện

* Khảo sát các bài toán ứng viên và chọn **kiểm duyệt văn bản độc hại tiếng Việt** — nhu cầu có thật, có bộ dữ liệu chuẩn công khai (ViHSD) và các API thương mại ưu tiên tiếng Anh phủ chưa tốt
* Chạy các thí nghiệm fine-tune đầu tiên trên XLM-RoBERTa-base bằng Google Colab và dựng baseline TF-IDF + Logistic Regression để đối chứng
* Đụng ngay giới hạn thực tế: không thành viên nào có GPU ở máy, còn runtime Colab miễn phí bị thu hồi sau vài giờ — điều này chặn việc train ở 3 epoch và chi phối toàn bộ kết quả về sau
* Vẽ kiến trúc mục tiêu, ước tính chi phí, đặt tiêu chí thành công (macro-F1 ≥ 0,85) và viết proposal
* Tham dự buổi cộng đồng FCAJ ngày 25/07 và trình bày Value Creation & Delivery Canvas của nhóm

#### Kết quả đạt được

* **Output:** proposal được duyệt kèm sơ đồ kiến trúc đầy đủ, cùng một model chạy được và một baseline để đo đối chứng
* Nhận diện sớm và trung thực về ràng buộc tài nguyên tính toán, được phân tích đầy đủ ở mục 5.3
