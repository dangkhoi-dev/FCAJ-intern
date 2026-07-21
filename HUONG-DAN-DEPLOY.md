# 🚀 Hướng dẫn deploy FCAJ Moderation lên AWS

> Dành cho **Quân + Đức** (task 7 backend, task 9 deploy & test).
> Source này do **Khôi + Quốc** chuẩn bị, đã cover xong **task 6** (đóng gói model → Lambda container) và **task 8** (frontend React).
> Region gợi ý: `ap-southeast-1` (Singapore). Bedrock Claude Haiku nếu region chưa hỗ trợ thì dùng `us-east-1` cho Bedrock (đã tách env `BEDROCK_REGION`).

## Kiến trúc

```
Amplify (React UI) → API Gateway (REST) → Lambda container (XLM-R ONNX INT8)
                                              ├─ confidence < 0.7 → Bedrock Claude Haiku (trọng tài)
                                              ├─ DynamoDB ModerationHistory (lịch sử)
                                              └─ CloudWatch (logs/metrics/alarm)
S3: chứa model artifacts (đã có từ task 5)
```

## Cấu trúc source

```
fcaj-moderation/
├── HUONG-DAN-DEPLOY.md        ← file này
├── README.md
├── model/                     ← Lambda container image (task 6 — Khôi)
│   ├── Dockerfile             ← base public.ecr.aws/lambda/python:3.11
│   ├── requirements.txt       ← onnxruntime + tokenizers (KHÔNG cần torch → image nhỏ)
│   ├── src/
│   │   ├── handler.py         ← routes: POST /moderate, GET /history (+ CORS, validate input)
│   │   ├── moderation.py      ← inference ONNX + clean_text giống notebook train + wordlist highlight
│   │   ├── bedrock_judge.py   ← gọi Claude Haiku khi confidence < 0.7
│   │   └── history.py         ← ghi/đọc DynamoDB
│   ├── artifacts/             ← ĐẶT model.onnx + tokenizer.json VÀO ĐÂY (xem artifacts/README.md)
│   ├── events/                ← event mẫu để test RIE
│   └── scripts/
│       ├── export_onnx.py     ← convert xlmr-vihsd-v0 → ONNX INT8 (nếu chưa có)
│       ├── test_local.sh      ← build + test bằng Lambda RIE
│       └── push_ecr.sh        ← build --platform amd64 + push ECR
└── frontend/                  ← React (Vite) UI (task 8 — Quốc)
```

---

## Bước 0 — Chuẩn bị

- Docker Desktop, AWS CLI v2 đã `aws configure` (dùng IAM Identity Center / SSO của Organization).
- Node.js ≥ 18 (cho frontend).
- Lấy model artifacts: tải `model.onnx` + `tokenizer.json` từ S3 bucket model (task 5) hoặc từ output notebook, đặt vào `model/artifacts/`. Chi tiết: `model/artifacts/README.md`.

## Bước 1 — Test local bằng Lambda RIE (đã script sẵn)

```bash
cd model
bash scripts/test_local.sh
```

Script tự build image, chạy container với RIE cổng 9000, bắn 4 event mẫu:
câu sạch, câu tục (teencode), input rỗng (mong đợi 400), input quá dài (mong đợi 400).
Kết quả mong đợi ví dụ:

```json
{"statusCode": 200, "body": "{\"label\": \"OFFENSIVE\", \"confidence\": 0.93, \"source\": \"model\", \"flaggedTerms\": [\"vcl\"], ...}"}
```

> Test local chạy với `ENABLE_BEDROCK=false`, `TABLE_NAME=` (rỗng) nên **không cần AWS credentials**.
> 📸 Chụp screenshot output cho báo cáo (task 9).

## Bước 2 — Push image lên ECR

```bash
cd model
AWS_ACCOUNT_ID=<12 số> AWS_REGION=ap-southeast-1 bash scripts/push_ecr.sh
```

Script tự tạo repo `fcaj-moderation` (nếu chưa có), login, build `--platform linux/amd64`, push tag `v1`. Ghi lại image URI in ra cuối script.

## Bước 3 — Tạo DynamoDB table

```bash
aws dynamodb create-table \
  --table-name ModerationHistory \
  --attribute-definitions \
      AttributeName=requestId,AttributeType=S \
      AttributeName=gsi1pk,AttributeType=S \
      AttributeName=gsi1sk,AttributeType=S \
  --key-schema AttributeName=requestId,KeyType=HASH \
  --global-secondary-indexes '[{
      "IndexName": "timestamp-index",
      "KeySchema": [
        {"AttributeName": "gsi1pk", "KeyType": "HASH"},
        {"AttributeName": "gsi1sk", "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "ALL"}
    }]' \
  --billing-mode PAY_PER_REQUEST \
  --region ap-southeast-1
```

> GSI theo timestamp: partition cố định `gsi1pk="HISTORY"`, sort key `gsi1sk=ISO timestamp` → query lịch sử mới nhất bằng 1 lệnh Query (không Scan, tiết kiệm chi phí — điểm cộng cho blog 3 của Đức 😄).

## Bước 4 — Tạo Lambda từ container image

Console → Lambda → Create function → **Container image**:

| Cấu hình | Giá trị |
|---|---|
| Function name | `fcaj-moderate` |
| Image URI | `<account>.dkr.ecr.ap-southeast-1.amazonaws.com/fcaj-moderation:v1` |
| Architecture | `x86_64` |
| Memory | **3008 MB** (XLM-R cần RAM + CPU tỉ lệ theo memory) |
| Timeout | **30 s** (cold start load model ~5-10s) |
| Ephemeral storage | mặc định 512 MB là đủ (model nằm trong image) |

Environment variables:

```
CONFIDENCE_THRESHOLD = 0.7
ENABLE_BEDROCK       = true
BEDROCK_MODEL_ID     = anthropic.claude-3-haiku-20240307-v1:0
BEDROCK_REGION       = us-east-1        # hoặc region đã enable model access
TABLE_NAME           = ModerationHistory
GSI_NAME             = timestamp-index
CORS_ORIGIN          = *                # sau khi có domain Amplify thì thay bằng domain đó
```

**IAM execution role** (least privilege — khớp thiết kế của Đức ở task 3), attach thêm policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DynamoDBHistory",
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem", "dynamodb:Query"],
      "Resource": [
        "arn:aws:dynamodb:ap-southeast-1:<ACCOUNT_ID>:table/ModerationHistory",
        "arn:aws:dynamodb:ap-southeast-1:<ACCOUNT_ID>:table/ModerationHistory/index/timestamp-index"
      ]
    },
    {
      "Sid": "BedrockJudge",
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel"],
      "Resource": "arn:aws:bedrock:*::foundation-model/anthropic.claude-3-haiku-*"
    }
  ]
}
```

> ⚠️ Bedrock: vào console Bedrock (region `BEDROCK_REGION`) → **Model access** → request access cho *Anthropic Claude 3 Haiku* trước, không thì Lambda sẽ lỗi AccessDenied (handler đã catch — sẽ fallback về kết quả model, xem CloudWatch log `[WARN] Bedrock fallback loi`).

Test nhanh trong console Lambda → Test tab, dán nội dung `model/events/offensive.json`.

## Bước 5 — API Gateway (REST API)

1. API Gateway → Create **REST API** → tên `fcaj-api`.
2. Tạo resource `/moderate` → method **POST** → Integration: **Lambda proxy** → `fcaj-moderate`.
3. Tạo resource `/history` → method **GET** → Lambda proxy → cùng function (handler tự route theo path).
4. Trên cả 2 resource: **Enable CORS** (Actions → Enable CORS, giữ mặc định — handler cũng đã tự trả CORS header nên OPTIONS mock của API GW hay của handler đều được).
5. **Deploy API** → stage `prod`. Ghi lại Invoke URL: `https://xxxx.execute-api.ap-southeast-1.amazonaws.com/prod`.

Test bằng curl:

```bash
API=https://xxxx.execute-api.ap-southeast-1.amazonaws.com/prod
curl -s -X POST $API/moderate -H 'Content-Type: application/json' \
  -d '{"text": "vcl thế, làm ăn như hạch"}' | python3 -m json.tool
curl -s "$API/history?limit=5" | python3 -m json.tool
```

## Bước 6 — Deploy frontend lên Amplify Hosting

Cách nhanh (deploy thủ công, không cần connect Git):

```bash
cd frontend
cp .env.example .env      # điền VITE_API_URL = Invoke URL ở bước 5 (không có / cuối)
npm install
npm run build             # ra thư mục dist/
cd dist && zip -r ../dist.zip . && cd ..
```

Amplify console → **Host web app** → *Deploy without Git provider* → upload `dist.zip`.

Cách chuẩn (CI/CD): push repo lên GitHub → Amplify connect repo, build settings:
`baseDirectory: frontend/dist`, build command `cd frontend && npm ci && npm run build`,
và khai báo biến môi trường `VITE_API_URL` trong Amplify (Environment variables).

Sau khi có domain Amplify → quay lại Lambda sửa `CORS_ORIGIN` thành domain đó (bảo mật hơn `*`).

## Bước 7 — Test end-to-end + monitoring (task 9)

Checklist test (chụp screenshot từng case cho báo cáo):

- [ ] Câu sạch → `CLEAN`, source `model`
- [ ] Câu tục thẳng ("Thằng này ngu như bò...") → `OFFENSIVE`, từ tục được **highlight đỏ** trên UI
- [ ] Teencode/lách từ ("vcl", "đ.m") → nhãn đúng, flaggedTerms có từ đó
- [ ] Câu mơ hồ (confidence < 0.7) → source `bedrock` + hiện "Lý do (Bedrock)"
- [ ] Tiếng Anh toxic ("You are so stupid...") → `OFFENSIVE` (zero-shot XLM-R)
- [ ] Input rỗng → thông báo lỗi tiếng Việt, không crash
- [ ] Input > 2000 ký tự → lỗi `TEXT_TOO_LONG`
- [ ] Tab Lịch sử hiện đúng các request vừa gửi (mới nhất trước)

CloudWatch:

```bash
# Alarm khi Lambda lỗi
aws cloudwatch put-metric-alarm --alarm-name fcaj-lambda-errors \
  --namespace AWS/Lambda --metric-name Errors \
  --dimensions Name=FunctionName,Value=fcaj-moderate \
  --statistic Sum --period 300 --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold --region ap-southeast-1
# Xem log
aws logs tail /aws/lambda/fcaj-moderate --follow --region ap-southeast-1
```

## Bước 8 — Clean-up (task 20, tránh phát sinh chi phí)

```bash
aws apigateway delete-rest-api --rest-api-id <id> --region ap-southeast-1
aws lambda delete-function --function-name fcaj-moderate --region ap-southeast-1
aws dynamodb delete-table --table-name ModerationHistory --region ap-southeast-1
aws ecr delete-repository --repository-name fcaj-moderation --force --region ap-southeast-1
# Amplify: console → App settings → Delete app
# S3 model bucket: giữ lại đến khi nộp báo cáo xong rồi hãy xoá
```

---

## Xử lý sự cố thường gặp

| Triệu chứng | Nguyên nhân / cách sửa |
|---|---|
| Lambda timeout lần gọi đầu | Cold start load model. Tăng timeout 30s, memory 3008MB. Warm-up bằng 1 request mồi. |
| `exec format error` khi Lambda chạy | Build image trên máy ARM (Mac M-series) thiếu `--platform linux/amd64` — dùng `push_ecr.sh` có sẵn flag. |
| `AccessDeniedException` từ Bedrock | Chưa request Model access, hoặc sai `BEDROCK_REGION`. Không chặn hệ thống — handler fallback về model. |
| CORS lỗi trên browser | Kiểm tra đã Deploy API sau khi Enable CORS; `CORS_ORIGIN` của Lambda khớp domain Amplify. |
| `Khong tim thay model.onnx` khi build/chạy | Chưa copy artifacts vào `model/artifacts/` — xem `model/artifacts/README.md`. |
| History rỗng dù đã gửi request | Kiểm tra IAM role có `dynamodb:PutItem`; xem CloudWatch log `[WARN] Ghi DynamoDB loi`. |

Có gì vướng ping Khôi (model/container) hoặc Quốc (frontend) trên group nhé!
