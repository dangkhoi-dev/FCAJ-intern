## A1. Cấu hình API URL cho frontend

Trên máy bạn (thư mục đã giải nén `fcaj-moderation/frontend`):

```bash
cd frontend
copy .env.example .env
```

Mở file `.env`, điền đúng 1 dòng:

```
VITE_API_URL=https://0hijbts7ph.execute-api.ap-southeast-1.amazonaws.com/prod
```

## A2. Build

```bash
npm install
npm run build
```

Ra thư mục `dist/`. Nén **nội dung bên trong** dist thành zip (index.html phải nằm ở gốc zip, không nằm trong folder con):

```powershell
cd dist
Compress-Archive -Path * -DestinationPath ..\dist.zip
```

## A3. Deploy lên Amplify (cách thủ công, nhanh nhất)

1. AWS Console → **Amplify** (region ap-southeast-1) → **Create new app** → **Deploy without Git**.
2. App name: `fcaj-moderation-ui`, kéo thả `dist.zip` → **Save and deploy**.
3. Chờ ~1 phút → nhận domain dạng `https://main.dxxxxxxxxx.amplifyapp.com`. Đây chính là **link website demo public** cho báo cáo.
   📸 *Screenshot: màn hình deploy thành công + website mở trên browser.*

## A4. Siết CORS (nên làm sau khi có domain)

Lambda `fcaj-moderate` → Configuration → Environment variables → sửa:

```
CORS_ORIGIN = https://main.dxxxxxxxxx.amplifyapp.com
```

(Đang để `*` vẫn chạy được, nhưng để domain cụ thể sẽ đẹp điểm bảo mật trong báo cáo.)

## A5. Khuyến nghị từ số liệu test của bạn (nên làm, 2 phút)

Screenshot cho thấy **Max memory used: 1011 MB / 1024 MB cấu hình** — sát trần, có nguy cơ OOM khi câu dài hoặc nhiều request cùng lúc, và cold start đang ~10s (curl đầu tiên latency 10426ms).

→ Lambda → Configuration → General configuration → **Memory: 2048 MB** (hoặc 3008 MB). Thêm memory = thêm CPU → inference nhanh hơn, cold start ngắn hơn. Ghi vào báo cáo phần "tối ưu" luôn: so sánh latency trước/sau là một hình rất đẹp.

---

# PHẦN B — TEST END-TO-END + BỘ SCREENSHOT

Mở website Amplify, test lần lượt và **chụp màn hình từng case** (đây chính là deliverable "bộ screenshot"):

| # | Test case | Kết quả mong đợi | 📸 |
|---|---|---|---|
| 1 | "Hôm nay thời tiết đẹp quá, chúc mọi người một ngày tốt lành!" | CLEAN, source model hoặc bedrock | ✔ |
| 2 | "vcl thế, làm ăn như hạch" (teencode) | OFFENSIVE, chữ "vcl" **highlight đỏ**, source bedrock + lý do | ✔ |
| 3 | "Thằng này ngu như bò" (chửi thẳng) | OFFENSIVE, "ngu" highlight | ✔ |
| 4 | "Đồ ng.u, cút đi" (lách từ) | OFFENSIVE/HATE | ✔ |
| 5 | "You are so stupid, shut up" (tiếng Anh — zero-shot) | OFFENSIVE | ✔ |
| 6 | Bấm Gửi khi ô trống / toàn dấu cách | Thông báo lỗi "Vui lòng nhập nội dung...", không crash | ✔ |
| 7 | Dán >2000 ký tự | Lỗi `TEXT_TOO_LONG` hiển thị đẹp | ✔ |
| 8 | Tab **Lịch sử** | Các câu vừa test hiện ra, mới nhất trước (chứng minh DynamoDB) | ✔ |
| 9 | Gửi 1 câu mơ hồ (vd "Mấy ô này tính lừa đảo hay gì mà làm ăn mập mờ thế nhỉ") | source **bedrock** + dòng "Lý do (Bedrock)" | ✔ |

Test bằng curl cho phần phụ lục báo cáo (bạn đã có sẵn 1 screenshot dạng này):

```bash
API=https://0hijbts7ph.execute-api.ap-southeast-1.amazonaws.com/prod
curl -s -X POST $API/moderate -H "Content-Type: application/json" -d "{\"text\": \"\"}"
curl -s "$API/history?limit=5"
```

---

# PHẦN C — CLOUDWATCH LOGS / METRICS / ALARM

1. **Logs**: CloudWatch → Log groups → `/aws/lambda/fcaj-moderate` → mở 1 log stream, 📸 chụp dòng REPORT (duration, memory).
2. **Metrics**: Lambda → Monitor tab → 📸 chụp biểu đồ Invocations + Duration + Error count sau khi chạy hết bộ test ở Phần B.
3. **Alarm** (chạy 1 lệnh):

```bash
aws cloudwatch put-metric-alarm --alarm-name fcaj-lambda-errors ^
  --namespace AWS/Lambda --metric-name Errors ^
  --dimensions Name=FunctionName,Value=fcaj-moderate ^
  --statistic Sum --period 300 --evaluation-periods 1 --threshold 1 ^
  --comparison-operator GreaterThanOrEqualToThreshold --region ap-southeast-1
```

📸 CloudWatch → Alarms → chụp alarm ở trạng thái OK. (Kèm screenshot Budget alarm $10 đã có từ task 1 là đủ trọn bộ monitoring.)

---
