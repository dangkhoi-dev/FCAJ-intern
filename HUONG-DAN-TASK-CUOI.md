# ✅ Hướng dẫn hoàn thành task cuối: Deploy & Test + Nộp báo cáo

> Trạng thái hiện tại (theo screenshot bạn gửi): Lambda `fcaj-moderate` chạy OK, Bedrock trọng tài hoạt động (source="bedrock" + lý do tiếng Việt), API Gateway đã live tại:
> **`https://0hijbts7ph.execute-api.ap-southeast-1.amazonaws.com/prod`**
>
> ⚠️ Link bạn được gửi qua Facebook là link bọc `l.facebook.com/l.php?u=...` kèm tham số `fbclid` — **KHÔNG dán nguyên link đó vào frontend**. Chỉ dùng đúng URL sạch ở trên (không có `/` cuối, không có `?fbclid=...`).

---

# PHẦN A — DEPLOY UI LÊN AMPLIFY HOSTING

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

# PHẦN D — CÀI HUGO ĐỂ XEM WEBSITE BÁO CÁO TRÊN MÁY (Windows)

Cách nhanh nhất bằng **winget** (PowerShell):

```powershell
winget install Hugo.Hugo.Extended
```

(Không có winget thì: tải file `hugo_extended_x.x.x_windows-amd64.zip` từ https://github.com/gohugoio/hugo/releases → giải nén `hugo.exe` vào `C:\Hugo\bin` → thêm vào PATH. Bản **Extended** nhé, theme cần nó.)

Mở PowerShell **mới**, kiểm tra và chạy:

```powershell
hugo version
cd D:\AWS\fcj-workshop-template
hugo server -D
```

Mở browser: **http://localhost:1313** — sửa file .md nào là trang tự reload ngay. Góc trên website có chọn English / Tiếng Việt để check song ngữ.

## Deploy website báo cáo (khi viết xong nội dung)

Repo đã có sẵn workflow `.github/workflows/hugo.yml` — chỉ cần:

1. Sửa `config.toml`: `baseURL = "https://<github-username>.github.io/<tên-repo>/"`
2. Push repo lên GitHub (branch `main`) → Actions tự build → vào Settings → Pages → chọn branch `gh-pages`.
3. Link `https://<github-username>.github.io/<tên-repo>/` là link nộp qua **Workshop submission** trong Self-service trên FCAJ Portal. **Hạn: hết ngày 31/07.**

---

# PHẦN E — KẾT QUẢ RÀ SOÁT BÁO CÁO THEO TIÊU CHÍ (hcm-rules.awsfcaj.com/3-project)

Thang điểm chính thức: Ý tưởng 1.0 | Kiến trúc 2.0 | Triển khai step-by-step 2.0 | Tài liệu & trình bày 0.5 | Đóng góp cá nhân 0.5.

## ✅ Phần ĐÃ ĐẠT

- **Trang chủ**: đã có thông tin sinh viên (tên, email, công ty, vị trí) song ngữ vi/en.
- **2-Proposal**: viết rất tốt, đầy đủ song ngữ, đúng cấu trúc (tóm tắt, vấn đề, kiến trúc, 9 dịch vụ AWS + lý do, timeline, ngân sách) — vượt yêu cầu "tối thiểu 3 dịch vụ AWS".
- **5-Workshop trang chính + 5.1 Overview**: đã viết theo đúng dự án moderation, có bảng dịch vụ + lý do chọn.

## ❌ Phần CHƯA ĐẠT — phải sửa trước 31/07 (xếp theo mức độ quan trọng)

**1. Mục 5.2 → 5.6 vẫn là nội dung template cũ (S3 VPC Endpoint) — NGHIÊM TRỌNG NHẤT, chiếm 2.0 điểm "Triển khai step-by-step".**
Cần viết lại toàn bộ theo dự án thật, gợi ý cấu trúc (đổi tên thư mục luôn cho sạch URL):

```
5.2-Prerequiste       → giữ tên, viết lại: AWS account/Organization, request Bedrock model access,
                        cài Docker + AWS CLI, IAM policy thật của dự án (lấy trong HUONG-DAN-DEPLOY.md)
5.3-S3-vpc            → 5.3-Train-Package-Model: train XLM-R trên Colab (screenshot notebook, confusion matrix),
                        export ONNX INT8, test local bằng Lambda RIE, push ECR
5.4-S3-onprem         → 5.4-Deploy-Backend: tạo DynamoDB table + GSI, tạo Lambda container (memory/timeout/env vars),
                        API Gateway REST + CORS, test trên console (bạn ĐÃ CÓ SẴN các screenshot này!)
5.5-Policy            → 5.5-Deploy-Frontend-Test: deploy Amplify (Phần A), bộ test e2e (Phần B), CloudWatch (Phần C)
5.6-Cleanup           → giữ tên, viết lại: xóa API GW, Lambda, DynamoDB, ECR, Amplify, alarm
                        (lệnh có sẵn trong HUONG-DAN-DEPLOY.md mục 8)
```

Lưu ý: đổi tên thư mục thì sửa cả link trong `5-Workshop/_index.md` + `_index.vi.md`, xóa các thư mục con cũ (`5.3.1-create-gwe`, `5.4.1-prepare`...), và **mỗi mục phải có đủ cặp `_index.md` (EN) + `_index.vi.md` (VI)**.

**2. Thiếu mục References (mục 9 theo yêu cầu).** Template không có sẵn — tạo thư mục `content/8-References/` với `_index.md` + `_index.vi.md` (weight: 8), trong đó đặt: link GitHub repo source code (fcaj-moderation), link website demo Amplify, link 3 bài blog đã đăng, video demo (nếu có), link dataset ViHSD/ViCTSD/Jigsaw, link tài liệu AWS. Yêu cầu ghi rõ "link repo github, source code, video demo để trong này".

**3. Mâu thuẫn tên model: Proposal + Workshop đang viết "PhoBERT/DistilBERT" nhưng notebook thực tế train XLM-RoBERTa.** Người chấm đọc code sẽ thấy lệch. Sửa thành: *"Ban đầu đề xuất PhoBERT (vi) + DistilBERT (en), sau chuyển sang XLM-RoBERTa-base vì một model xử lý được đa ngôn ngữ (zero-shot) và đơn giản hóa triển khai"* — vừa đúng thực tế vừa thể hiện tư duy kỹ thuật.

**4. Các mục còn nguyên warning template "không sao chép nguyên văn":** 1-Worklog (12 tuần), 3-BlogsPosted, 4-EventParticipated, 6-Self-evaluation, 7-Feedback. Phải viết lại bằng nội dung thật + **xóa hết các block `{{% notice warning %}}`**. BlogsPosted: dán tiêu đề + tóm tắt + link 3 bài đã đăng group AWS Study Group (dùng nội dung trong `Caption-Facebook-FCAJ.txt` làm tóm tắt rất nhanh).

**5. Placeholder chưa điền ở trang chủ:** số điện thoại, trường, ngành, lớp, thời gian thực tập ([FILL]/[ĐIỀN] cả bản en lẫn vi) + thay ảnh `static/images/avatar.png` bằng ảnh của bạn.

**6. `config.toml`:** `baseURL` còn là `workshop-sample.awsfcaj.com` → đổi thành URL GitHub Pages thật; `author` còn là `thienlh@thienlu.com` → đổi email của bạn; title có thể đổi thành "FCAJ — Toxic Text Moderation Platform".

**7. File đính kèm (tiêu chí "file đính kèm nếu phù hợp"):** đưa `Dockerfile`, `handler.py` snippet, IAM policy JSON vào code block trong mục 5.4; commit repo `fcaj-moderation` lên GitHub và link trong References — vừa đạt tiêu chí vừa xong luôn việc "backup code lên GitHub" của task 20.

## 📸 Danh sách hình ảnh cần bạn bổ sung vào `static/images/`

Bỏ vào folder `D:\AWS\fcj-workshop-template\static\images\` (mình liệt kê theo mục, tên gợi ý):

- `2-Proposal/architecture.png` — sơ đồ draw.io của Đức & Quân (đang ghi "cập nhật sau") — **quan trọng, tiêu chí bắt buộc "sơ đồ kiến trúc"**
- `5-Workshop/architecture.png` — cùng sơ đồ trên (hoặc bản chi tiết hơn)
- `5-Workshop/5.3/` — screenshot notebook Colab: phân bố nhãn, confusion matrix, bảng so sánh baseline vs XLM-R, test RIE local, push ECR thành công
- `5-Workshop/5.4/` — screenshot bạn ĐÃ CÓ: Lambda test console (3 cái bạn vừa gửi mình), DynamoDB table + items, API Gateway stages, Bedrock model access
- `5-Workshop/5.5/` — screenshot Amplify deploy + website demo từng test case (Phần B) + CloudWatch metrics/alarm (Phần C)
- `5-Workshop/5.6/` — screenshot xóa resource (chụp lúc clean-up, ĐỪNG clean-up trước khi chụp xong demo!)
- `avatar.png` — ảnh đại diện của bạn (thay file mặc định)
- 3-BlogsPosted: screenshot 3 post trên group Facebook

---

# THỨ TỰ LÀM GỢI Ý (deadline 31/07)

1. **Hôm nay**: Phần A (Amplify) + B (test, chụp screenshot) + C (alarm) → xong task 9 trong bảng.
2. **24–28/07**: viết lại 5.2–5.6 song ngữ + References + Worklog/Blogs/Events/Self-eval/Feedback, bỏ ảnh vào, preview bằng `hugo server`.
3. **29–30/07**: push GitHub → GitHub Pages, review chéo cả nhóm, nộp link qua Workshop submission trên FCAJ Portal.
4. **Sau khi nộp + chụp đủ screenshot**: clean-up AWS resources (mục 8 trong HUONG-DAN-DEPLOY.md).

Cần mình viết luôn nội dung song ngữ cho các mục 5.2–5.6 và 8-References thì cứ nói — screenshot nào có rồi bạn bỏ vào folder `static/images/` theo cấu trúc trên là mình ráp được ngay.
