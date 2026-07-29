---
title: "Worklog Tuần 7"
date: 2026-07-13
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

### Tuần 7: Lab — Storage & S3

**Thời gian:** 22/06/2026 – 28/06/2026

#### Mục tiêu

* Dùng S3 như một kho phân phối artefact thật sự, không chỉ là chỗ quăng file
* Thực hành các mẫu kiểm soát truy cập mà project sẽ cần

#### Công việc đã thực hiện

* Tạo bucket có versioning và mã hoá mặc định; upload và tải object qua console, qua CLI và qua presigned URL
* Viết bucket policy rồi đối chiếu với IAM identity policy để xem cái nào thắng khi xung đột
* Đặt lifecycle rule chuyển object sang Infrequent Access rồi hết hạn, và xác nhận các bước chuyển trên console
* Bật static website hosting trên một bucket và phục vụ một trang test

#### Kết quả đạt được

* **Output:** một bucket S3 mà từ đây trở đi nhóm dùng làm điểm phân phối các artefact model đã export (`model.onnx` + `tokenizer.json`)
* Thành thạo presigned URL và bucket policy — chính là cơ chế dùng để chia file model trong nhóm mà không phải public bất cứ thứ gì
