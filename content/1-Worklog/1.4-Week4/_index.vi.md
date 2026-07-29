---
title: "Worklog Tuần 4"
date: 2026-07-13
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

### Tuần 4: Lab — Nền tảng tài khoản & quản trị

**Thời gian:** 01/06/2026 – 07/06/2026

#### Mục tiêu

* Biến lý thuyết của tuần 1–3 thành một tài khoản AWS thật, cấu hình an toàn
* Tạo ra các thành phần quản trị mà project sẽ dựng lên trên đó

#### Công việc đã thực hiện

* Tạo AWS Organization và dựng cấu trúc tổ chức cho nhóm
* Thiết lập IAM Identity Center, tạo permission set riêng cho từng thành viên và ngừng hẳn việc dùng tài khoản root; bật MFA toàn bộ
* Tạo AWS Budget kèm cảnh báo email để mọi chi phí bất thường lộ ra trong vòng vài giờ
* Xin và bật quyền truy cập model Anthropic Claude Haiku trên Amazon Bedrock
* Cài và cấu hình AWS CLI với profile SSO, kiểm chứng bằng `aws sts get-caller-identity`

#### Kết quả đạt được

* **Output:** một thiết lập nhiều tài khoản có quản trị — Organization, permission set của Identity Center, cảnh báo ngân sách và quyền model Bedrock, tất cả đã chụp màn hình đưa vào mục 5.2 của báo cáo này
* Tài khoản đủ an toàn để bốn người cùng dùng mà không ai phải chia sẻ credential
