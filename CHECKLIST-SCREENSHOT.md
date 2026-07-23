# 📸 Checklist chụp screenshot cho báo cáo — kèm hướng dẫn từng tấm

> Cách dùng: chụp bằng **Win + Shift + S** (chọn vùng) hoặc Alt + PrtSc (cả cửa sổ).
> Lưu đúng **tên file + thư mục** ghi bên dưới (trong `D:\AWS\fcj-workshop-template\static\images\`) — tên khớp sẵn để lát mình viết nội dung 5.2–5.6 là ảnh tự hiện, không phải sửa link.
>
> Quy tắc chung cho mọi tấm:
> - Console AWS để **tiếng Anh**, browser zoom 100%, chụp đủ breadcrumb phía trên (để thấy đang ở service nào, region nào).
> - **Che Account ID 12 số + email cá nhân** trước khi bỏ vào báo cáo (Paint → bôi đen, hoặc tool blur của Snipping Tool).
> - Chụp XONG HẾT các phần khác rồi mới làm 5.6 Clean-up (xóa rồi là hết chụp demo).

---

## 🟥 PHẦN PHẢI HỐI QUÂN + ĐỨC (hạ tầng / backend — account của họ mới có quyền xem)

### Đức — sơ đồ + hạ tầng (task 3 + 5 của Đức)

| # | File lưu vào `static/images/` | Cách chụp |
|---|---|---|
| D1 | `2-Proposal/architecture.png` (và copy thành `5-Workshop/architecture.png`) | **QUAN TRỌNG NHẤT — tiêu chí bắt buộc.** Đức mở file `architect.drawio` (đã có từ task 3) trên draw.io → File → Export as → PNG → Zoom 200%, bật "Transparent background" tắt, border 10 → tải về. KHÔNG chụp màn hình draw.io (mờ), phải export. |
| D2 | `5-Workshop/5.4/dynamodb-table.png` | Console → DynamoDB → Tables → `ModerationHistory` → tab **Overview**: thấy tên bảng, PK `requestId`, billing mode |
| D3 | `5-Workshop/5.4/dynamodb-gsi.png` | Cùng trang → tab **Indexes**: thấy GSI `timestamp-index` (gsi1pk / gsi1sk) |
| D4 | `5-Workshop/5.4/dynamodb-items.png` | Tab **Explore table items** SAU khi đã test vài câu: thấy các record label/confidence/source. Che bớt text nhạy cảm nếu có |
| D5 | `5-Workshop/5.4/iam-role-policy.png` | Console → IAM → Roles → role của Lambda `fcaj-moderate` → mở policy → tab JSON (thấy dynamodb:PutItem/Query + bedrock:InvokeModel) |
| D6 | `5-Workshop/5.3/ecr-repo.png` | Console → ECR → Repositories → `fcaj-moderation` → thấy image tag `v1`, size, pushed date |
| D7 | `2-Proposal/` (tùy) | Xin luôn Đức file "bảng lý do chọn service + IAM Least Privilege" (task 3) để dán vào mục kiến trúc |

### Quân — Organization + backend (task 1 + 7 của Quân)

| # | File | Cách chụp |
|---|---|---|
| Q1 | `5-Workshop/5.2/organization.png` | Console (account management) → AWS Organizations → **AWS accounts**: thấy cây Organization + 4 member accounts. **CHE EMAIL + ACCOUNT ID** |
| Q2 | `5-Workshop/5.2/identity-center.png` | IAM Identity Center → Users: danh sách 4 user (che email) |
| Q3 | `5-Workshop/5.2/budget-alarm.png` | Billing → Budgets → budget $10: thấy threshold + alert email. Kèm 1 tấm CloudWatch billing alarm nếu có |
| Q4 | `5-Workshop/5.2/bedrock-access.png` | Console → Bedrock (đúng region đã bật, ví dụ us-east-1) → **Model access**: dòng *Claude 3 Haiku* trạng thái **Access granted** |
| Q5 | `5-Workshop/5.4/lambda-overview.png` | Lambda → `fcaj-moderate` → trang chính: thấy "Package type: Image", memory, timeout (chụp SAU khi đã tăng memory lên 2048MB) |
| Q6 | `5-Workshop/5.4/lambda-env.png` | Lambda → Configuration → **Environment variables**: thấy CONFIDENCE_THRESHOLD, TABLE_NAME, BEDROCK_MODEL_ID... |
| Q7 | `5-Workshop/5.4/lambda-test-clean.png`, `lambda-test-offensive.png`, `lambda-test-bedrock.png` | ✅ **ĐÃ CÓ** — chính là 3 screenshot test console Quân gửi hôm trước (câu sạch/HATE, câu vcl → bedrock, câu mơ hồ → bedrock). Bảo Quân gửi bản gốc PNG |
| Q8 | `5-Workshop/5.4/apigw-resources.png` | API Gateway → `fcaj-api` → **Resources**: thấy cây `/moderate` (POST) + `/history` (GET) |
| Q9 | `5-Workshop/5.4/apigw-stage.png` | API Gateway → Stages → `prod`: thấy **Invoke URL** `https://0hijbts7ph...` |
| Q10 | `5-Workshop/5.4/curl-test.png` | ✅ **ĐÃ CÓ** — screenshot terminal curl của Quân (câu "vcl thế..." trả JSON bedrock). Xin bản gốc |

### Blog (sau khi đăng — task 19)

| # | File | Ai |
|---|---|---|
| B1-B3 | `3-BlogsPosted/blog1.png`, `blog2.png`, `blog3.png` | Chụp post trên group AWS Study Group sau khi đăng: Quân (blog 1), **Khôi — bạn** (blog 2), Đức (blog 3) |

---

## 🟩 PHẦN CỦA BẠN (KHÔI) + QUỐC — model, local test, UI

### Khôi — mục 5.3 Train & Package Model

| # | File | Cách chụp |
|---|---|---|
| K1 | `5-Workshop/5.3/colab-gpu.png` | Mở notebook `train_text_classifier.ipynb` trên Colab → cell đầu in ra `GPU: Tesla T4` → chụp cell + output |
| K2 | `5-Workshop/5.3/label-distribution.png` | Cell EDA mục 3: biểu đồ cột "Phan phoi nhan (train)" ~82% CLEAN. Nếu notebook đã chạy từ trước thì output còn lưu sẵn trong file — chỉ cần mở ra chụp, KHÔNG cần chạy lại |
| K3 | `5-Workshop/5.3/training-log.png` | Cell fine-tune mục 6: bảng log các epoch (loss, macro_f1 tăng dần) |
| K4 | `5-Workshop/5.3/confusion-matrix.png` | Cell mục 7: heatmap confusion matrix — chụp riêng hình cho nét |
| K5 | `5-Workshop/5.3/compare-baseline.png` | Cell mục 7: bảng so sánh Baseline TF-IDF vs XLM-R (F1 từng nhãn + Macro-F1) |
| K6 | `5-Workshop/5.3/demo-predict.png` | Cell mục 8: output 10 câu demo (có câu tiếng Anh/Trung zero-shot) — điểm cộng lớn |
| K7 | `5-Workshop/5.3/onnx-export.png` | Cell mục 10: output export ONNX + INT8, dòng benchmark `ONNX INT8 CPU: p50=... p95=...` |
| K8 | `5-Workshop/5.3/rie-test.png` | Trên máy bạn: `cd fcaj-moderation\model` → chạy `bash scripts/test_local.sh` (hoặc thủ công `docker run -p 9000:8080 ...` rồi curl `http://localhost:9000/2015-03-31/functions/function/invocations`) → chụp terminal thấy JSON statusCode 200 + label. Nếu Windows không có bash: chạy từng lệnh docker build / docker run / curl trong PowerShell |
| K9 | `5-Workshop/5.3/docker-push.png` | Terminal lúc chạy `push_ecr.sh`: các dòng `Pushed` + dòng cuối `XONG: ...dkr.ecr...v1` (nếu Quân là người push thì xin Quân) |

### Khôi + Quốc — mục 5.5 Frontend & Test (làm ngay sau khi fix xong 404 Amplify)

| # | File | Cách chụp |
|---|---|---|
| K10 | `5-Workshop/5.5/amplify-deployed.png` | Amplify console → app `fcaj-moderation-ui` → thấy branch Deployed + domain (bản đã fix 404) |
| K11 | `5-Workshop/5.5/ui-clean.png` | Website demo: gửi câu sạch → badge CLEAN xanh |
| K12 | `5-Workshop/5.5/ui-offensive-highlight.png` | Gửi "vcl thế, làm ăn như hạch" → badge OFFENSIVE + chữ **vcl highlight đỏ** — tấm ảnh "đinh" của cả báo cáo |
| K13 | `5-Workshop/5.5/ui-bedrock.png` | Gửi câu mơ hồ ("Mấy ô này tính lừa đảo hay gì...") → tag ⚖️ Bedrock + dòng "Lý do (Bedrock)" |
| K14 | `5-Workshop/5.5/ui-english.png` | "You are so stupid, shut up" → OFFENSIVE (zero-shot) |
| K15 | `5-Workshop/5.5/ui-error-empty.png` | Bấm Gửi khi ô trống → thông báo lỗi tiếng Việt |
| K16 | `5-Workshop/5.5/ui-error-toolong.png` | Dán >2000 ký tự (mở Notepad gõ 1 dòng rồi Ctrl+A Ctrl+C dán nhiều lần) → lỗi TEXT_TOO_LONG |
| K17 | `5-Workshop/5.5/ui-history.png` | Tab **Lịch sử**: các câu vừa test, mới nhất trước |
| K18 | `5-Workshop/5.5/cloudwatch-logs.png` | CloudWatch → Log groups → `/aws/lambda/fcaj-moderate` → mở log stream mới nhất → chụp có dòng `REPORT ... Duration ... Max Memory Used` |
| K19 | `5-Workshop/5.5/cloudwatch-metrics.png` | Lambda → tab **Monitor**: biểu đồ Invocations + Duration + Errors sau khi test xong cả bộ (đợi 2–3 phút cho metric hiện) |
| K20 | `5-Workshop/5.5/cloudwatch-alarm.png` | Sau khi tạo alarm (lệnh trong HUONG-DAN-TASK-CUOI Phần C) → CloudWatch → All alarms → `fcaj-lambda-errors` trạng thái **OK** (đợi vài phút cho đủ datapoint) |
| K21 | `avatar.png` (ghi đè file có sẵn) | Ảnh chân dung của bạn, crop vuông ~400x400 |

### Cả nhóm — mục 5.6 Clean-up (LÀM CUỐI CÙNG, ~30/07, sau khi đã chụp đủ mọi thứ trên)

| # | File | Cách chụp |
|---|---|---|
| C1 | `5-Workshop/5.6/delete-amplify.png` | Amplify → App settings → General → Delete app → chụp hộp thoại xác nhận |
| C2 | `5-Workshop/5.6/delete-apigw.png` | API Gateway → chọn `fcaj-api` → Actions → Delete |
| C3 | `5-Workshop/5.6/delete-lambda.png` | Lambda → chọn `fcaj-moderate` → Actions → Delete |
| C4 | `5-Workshop/5.6/delete-dynamodb.png` | DynamoDB → chọn bảng → Delete table |
| C5 | `5-Workshop/5.6/delete-ecr.png` | ECR → chọn repo → Delete |
| C6 | `5-Workshop/5.6/billing-final.png` | Billing → Bills: tổng chi phí cuối dự án (con số nhỏ = điểm cộng "tối ưu chi phí") |

Sau khi chụp C1–C6 và xóa thật → bỏ ảnh vào folder → build lại site → push GitHub → nộp link.

---

## 📨 Tin nhắn mẫu gửi group để hối Quân + Đức (copy gửi luôn)

> **@Đức**: xuất giùm tao file `architect.drawio` ra PNG (File → Export as PNG, zoom 200%) gửi vào đây — thiếu sơ đồ kiến trúc là mất điểm tiêu chí bắt buộc đó. Kèm luôn: screenshot DynamoDB table + tab Indexes + items, IAM role policy JSON của Lambda, ECR repo có tag v1, và file bảng lý do chọn service hồi task 3.
>
> **@Quân**: gửi tao bản gốc 3 screenshot test Lambda console + screenshot curl hôm trước. Chụp thêm: AWS Organizations (che email/account id), IAM Identity Center users, Budget alarm $10, Bedrock Model access (Claude Haiku - Access granted), Lambda overview + env vars (sau khi tăng memory 2048), API Gateway resources + stage prod. Danh sách chi tiết trong file CHECKLIST-SCREENSHOT.md tao gửi.
>
> Deadline gom ảnh: **28/07** để tao còn ráp vào site + review trước khi nộp 31/07.

---

## Tổng kết số lượng

- Đức: **7 mục** (quan trọng nhất: sơ đồ kiến trúc D1)
- Quân: **10 mục** (2 cái đã có sẵn Q7, Q10)
- Khôi (bạn): **~15 mục** — K1–K7 chỉ là mở notebook chụp output có sẵn, nhanh; K10–K20 làm một lượt sau khi fix 404 Amplify (~30 phút)
- Cả nhóm: 6 mục clean-up làm cuối

Gom đủ ảnh bỏ vào `static/images/` theo đúng tên trên → báo mình, mình viết luôn nội dung song ngữ 5.2–5.6 + 8-References khớp với bộ ảnh này.
