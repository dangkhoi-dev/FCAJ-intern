---
title: "Deploy frontend & kiểm thử"
date: 2026-07-13
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

Mục này deploy giao diện React lên Amplify Hosting, chạy bộ kiểm thử end-to-end (bao gồm các test case lỗi) và bật giám sát CloudWatch.

#### 1. Build frontend với API URL thật

Frontend là ứng dụng React (Vite): khung chat nhập câu → gọi API → hiển thị nhãn theo màu + confidence + nguồn quyết định (🤖 Model / ⚖️ Bedrock) + **highlight đỏ từ vi phạm** + tab lịch sử đọc từ DynamoDB.

```bash
cd frontend
cp .env.example .env
# Mở .env, điền Invoke URL từ mục 5.4 (KHÔNG có dấu / cuối):
# VITE_API_URL=https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/prod
npm install
npm run build      # ra thư mục dist/
```

#### 2. Deploy lên Amplify Hosting

Nhóm dùng cách deploy thủ công (không cần connect Git):

1. Console → **Amplify** → **Create new app** → **Deploy without Git**.
2. App name `fcaj-moderation-ui` → kéo thả **thư mục `dist`** (hoặc file zip nén **nội dung bên trong** dist) → **Save and deploy**.
3. Sau ~1 phút nhận domain public dạng `https://<branch>.<app-id>.amplifyapp.com`.

{{% notice warning %}}
Nếu nén zip, `index.html` phải nằm ngay ở **gốc** file zip. Nén nguyên thư mục `dist` từ bên ngoài sẽ tạo một lớp thư mục thừa → Amplify trả **HTTP 404**. Nhóm đã gặp đúng lỗi này và sửa bằng cách `cd dist` rồi mới nén `*`.
{{% /notice %}}

![Amplify deployed](/images/5-Workshop/5.5/amplify-deployed.png)

Sau khi có domain, quay lại Lambda cập nhật `CORS_ORIGIN` thành domain Amplify (siết chặt hơn `*`).

#### 3. Bộ kiểm thử end-to-end

Chạy trên website public, bao phủ cả trường hợp thành công lẫn lỗi:

| # | Test case | Kết quả mong đợi | Kết quả |
|---|---|---|---|
| 1 | Câu sạch: "Hôm nay thời tiết đẹp quá..." | CLEAN | ✅ |
| 2 | Teencode: "vcl thế, làm ăn như hạch" | OFFENSIVE, "vcl" highlight, nguồn Bedrock | ✅ |
| 3 | Chửi thẳng: "Thằng này ngu như bò" | OFFENSIVE, "ngu" highlight | ✅ |
| 4 | Lách từ: "Đồ ng.u, cút đi" | OFFENSIVE/HATE | ✅ |
| 5 | Tiếng Anh: "You are so stupid, shut up" | OFFENSIVE (zero-shot) | ✅ |
| 6 | Input rỗng | Nút **Analyze** bị disable, request không hề được gửi đi; gọi thẳng API thì trả về `EMPTY_TEXT` | ✅ |
| 7 | Input > 2000 ký tự | Ô nhập bị chặn ở 2.000 ký tự bằng `maxLength`; gọi thẳng API thì trả về `TEXT_TOO_LONG` | ✅ |
| 8 | Tab Lịch sử | Hiện các câu vừa test, mới nhất trước | ✅ |
| 9 | Câu mơ hồ (confidence < 0.7) | Nguồn ⚖️ Bedrock + dòng lý do | ✅ |

Câu sạch được nhận diện đúng:

![UI câu sạch](/images/5-Workshop/5.5/ui-clean.png)

Từ tục teencode bị highlight đỏ, nhãn OFFENSIVE:

![UI highlight từ tục](/images/5-Workshop/5.5/ui-offensive-highlight.png)

Trường hợp model không chắc chắn — Bedrock phân xử kèm lý do:

![UI Bedrock phân xử](/images/5-Workshop/5.5/ui-bedrock.png)

Zero-shot với tiếng Anh:

![UI tiếng Anh](/images/5-Workshop/5.5/ui-english.png)

**Xử lý input lỗi (chặn ngay ở tầng UI, không cần ảnh minh chứng).** Case 6 và 7 được xử lý *trước khi* request được gửi đi nên không tạo ra màn hình lỗi nào để chụp: nút **Phân tích** bị disable khi ô nhập còn rỗng, tức là không thể gửi input rỗng; và ô textarea đặt `maxLength={2000}` nên trình duyệt ngừng nhận ký tự đúng tại mốc 2.000, không có cách nào vượt giới hạn từ giao diện. Bộ đếm ký tự dưới ô nhập chuyển sang màu hổ phách khi qua 1.800 ký tự để cảnh báo sớm. Backend vẫn kiểm tra độc lập cả hai điều kiện này (`EMPTY_TEXT` và `TEXT_TOO_LONG` trong `handler.py`) để một request gọi thẳng vào API — bỏ qua frontend — cũng bị từ chối; nhánh server-side đó chính là phần được bộ test local bằng RIE kiểm tra.

Lịch sử kiểm duyệt đọc từ DynamoDB qua GSI:

![UI lịch sử](/images/5-Workshop/5.5/ui-history.png)

#### 4. Giám sát với CloudWatch

**Logs** — mỗi request có dòng `REPORT` với duration và memory thực dùng; các cảnh báo (Bedrock lỗi, DynamoDB lỗi) được handler chủ động ghi log với prefix `[WARN]`:

![CloudWatch logs](/images/5-Workshop/5.5/cloudwatch-logs.png)

**Metrics** — Invocations, Duration, Errors của Lambda sau bộ test:

![CloudWatch metrics](/images/5-Workshop/5.5/cloudwatch-metrics.png)

**Alarm** — cảnh báo khi Lambda có lỗi:

```bash
aws cloudwatch put-metric-alarm --alarm-name fcaj-lambda-errors \
  --namespace AWS/Lambda --metric-name Errors \
  --dimensions Name=FunctionName,Value=fcaj-moderate \
  --statistic Sum --period 300 --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold --region ap-southeast-1
```

![CloudWatch alarm](/images/5-Workshop/5.5/cloudwatch-alarm.png)

Hệ thống đã chạy end-to-end với giám sát đầy đủ. Bước cuối cùng: dọn dẹp tài nguyên.
