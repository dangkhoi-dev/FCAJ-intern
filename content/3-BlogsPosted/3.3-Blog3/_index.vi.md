---
title: "Blog 3"
date: 2026-07-13
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

# Thiết kế bảng DynamoDB cho một hệ thống AI ghi log: access pattern, GSI và on-demand

**Người viết & đăng bài:** Trần Quân &emsp;|&emsp; **Đăng tại:** group Facebook AWS Study Group

**Link bài đăng:** đăng ngày 28/07/2026 trong group [AWS Study Group VN](https://www.facebook.com/groups/awsstudygroupfcj) — đang chờ quản trị viên duyệt tại thời điểm nộp báo cáo

#### Tóm tắt

Khi làm một hệ thống AI trên AWS, phần ai cũng tập trung vào là model. Bài viết cho rằng chính phần ngay sau đó — kết quả đi về đâu và lấy ngược ra bằng cách nào — mới là chỗ rất dễ làm sai, đúng vì nó trông có vẻ đơn giản.

Bối cảnh: mỗi lần gọi API AI sẽ sinh ra một bản ghi gồm nội dung đầu vào, nhãn dự đoán, độ tin cậy, có phải gọi thêm LLM hay không, và độ trễ. Giao diện cần một màn hình "Lịch sử" hiển thị N bản ghi mới nhất.

**Nguyên tắc số 1: thiết kế bảng từ CÂU TRUY VẤN, không phải từ THỰC THỂ.** Đây là điểm khác biệt lớn nhất giữa DynamoDB và SQL. Với SQL, bạn dựng bảng theo thực thể rồi muốn query kiểu gì cũng được. Với DynamoDB thì ngược lại — phải liệt kê trước các access pattern, rồi mới thiết kế key theo đúng các pattern đó.

Với workload này có đúng hai access pattern:

1. Lấy một bản ghi theo id.
2. Liệt kê N bản ghi mới nhất, sắp xếp mới trước cũ sau.

Pattern 1 giải quyết bằng partition key là `id` (UUID). Pattern 2 mới là chỗ đáng nói, và bài viết phân tích **cái bẫy `Scan` rồi sort trong code** — cách này đọc toàn bộ bảng mỗi lần tải trang, càng thêm bản ghi càng đắt, và hỏng âm thầm ngay khi kết quả vượt giới hạn 1 MB mỗi trang. Lời giải đúng là một **Global Secondary Index** với partition key cố định và timestamp làm sort key, query theo thứ tự giảm dần kèm `Limit`.

Bài khép lại ở **on-demand so với provisioned capacity**: với một workload demo giật cục, khó đoán, on-demand gần như không tốn gì lúc rảnh, còn provisioned thì vẫn tính tiền dù có traffic hay không.

#### Ảnh chụp bài đăng

![Blog 3 trên AWS Study Group](/images/3-BlogsPosted/blog3.png)
