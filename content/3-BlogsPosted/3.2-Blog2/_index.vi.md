---
title: "Blog 2"
date: 2026-07-13
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

# Từ React tới người dùng: Amplify Hosting, API Gateway và CloudWatch cho một ứng dụng AI serverless

**Người viết & đăng bài:** Lê Đức &emsp;|&emsp; **Đăng tại:** group Facebook AWS Study Group

**Link bài đăng:** đăng ngày 28/07/2026 trong group [AWS Study Group VN](https://www.facebook.com/groups/awsstudygroupfcj) — đang chờ quản trị viên duyệt tại thời điểm nộp báo cáo

#### Tóm tắt

Bài viết bắt đầu đúng ở chỗ mà phần lớn bài về AI dừng lại. Việc model chạy đúng trên máy mình mới là một nửa công việc; nửa còn lại là khâu đưa nó tới tay người dùng — host giao diện ở đâu, cho nó nói chuyện với backend thế nào mà không vỡ, và làm sao *nhìn thấy* được chuyện gì đang xảy ra khi nó vỡ.

Bài đi qua ba phần theo thứ tự:

1. **Amplify Hosting.** Build thẳng app React (Vite) từ repository Git — một lần push là một lần deploy, không phải tự upload thư mục `dist/`, phục vụ qua HTTPS trên CDN, không có server nào để vá và cũng không có bucket nào để lỡ tay cấu hình nhầm quyền public. Hai lưu ý thực tế: để endpoint API trong biến môi trường lúc build (`VITE_API_URL`) chứ đừng hard-code trong source, đổi backend thì chỉ cần đổi biến rồi build lại; và tận dụng môi trường theo nhánh của Amplify để có một URL staging chạy song song với bản chính thức.
2. **Lỗi CORS mà ai cũng gặp ngày đầu tiên.** Triệu chứng kinh điển: gọi API bằng `curl` thì chạy ngon lành, nhưng mở trên trình duyệt thì fail, console báo lỗi CORS. CORS là cơ chế của *trình duyệt*, không phải của server — `curl` không phải trình duyệt nên nó không bao giờ quan tâm. Để đi qua được, phía API Gateway cần đúng hai thứ: một method `OPTIONS` (preflight) trả về các header `Access-Control-Allow-Origin` / `-Headers` / `-Methods`, **và** chính các header đó phải xuất hiện lại trong response `POST` thật. Với Lambda proxy integration, các header CORS phải do chính response của function trả ra — API Gateway sẽ không tự thêm giúp bạn.
3. **Một cái bẫy khi deploy bằng zip:** file `index.html` phải nằm ngay gốc file zip. Nén nguyên thư mục *chứa* `dist` sẽ tạo thêm một lớp thư mục thừa, và Amplify trả về HTTP 404 dù báo deploy "thành công". Cứ `cd` vào `dist` rồi mới nén là xong.

#### Ảnh chụp bài đăng

![Blog 2 trên AWS Study Group](/images/3-BlogsPosted/blog2.png)
