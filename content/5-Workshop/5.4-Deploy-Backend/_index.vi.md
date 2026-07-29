---
title: "Triển khai backend"
date: 2026-07-13
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

Mục này triển khai toàn bộ tầng backend: bảng DynamoDB lưu lịch sử → Lambda function từ container image → REST API qua API Gateway → tích hợp Bedrock làm trọng tài.

#### 1. Tạo bảng DynamoDB `ModerationHistory`

Thiết kế key: partition key `requestId`; để truy vấn "N bản ghi mới nhất" hiệu quả, nhóm tạo **GSI `timestamp-index`** với partition key cố định `gsi1pk = "HISTORY"` và sort key `gsi1sk = ISO timestamp`. Nhờ vậy việc lấy lịch sử chỉ tốn **1 lệnh Query** đọc đúng N item (không bao giờ phải Scan cả bảng — quan trọng cho cả chi phí lẫn độ trễ).

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

![Bảng DynamoDB](/images/5-Workshop/5.4/dynamodb-table.png)

![GSI timestamp-index](/images/5-Workshop/5.4/dynamodb-gsi.png)

#### 2. Tạo Lambda function từ container image

Console → Lambda → **Create function** → **Container image**:

| Cấu hình | Giá trị | Ghi chú |
|---|---|---|
| Function name | `fcaj-moderate` | |
| Image URI | `<ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/fcaj-moderation:v1` | image từ mục 5.3 |
| Architecture | `x86_64` | khớp `--platform linux/amd64` |
| Memory | **2048 MB** | Lambda cấp CPU tỉ lệ theo memory — thêm memory = inference nhanh hơn |
| Timeout | **30 s** | cold start cần load model ~5–10s |

{{% notice info %}}
Lần cấu hình đầu nhóm để 1024 MB và quan sát **Max memory used = 1011 MB** — sát trần, có nguy cơ out-of-memory. Sau khi tăng lên 2048 MB, hệ thống ổn định và latency cải thiện đáng kể. Đây là bài học thực tế về việc benchmark nhiều mức memory thay vì để mặc định.
{{% /notice %}}

Environment variables:

```
CONFIDENCE_THRESHOLD = 0.7
ENABLE_BEDROCK       = true
BEDROCK_MODEL_ID     = anthropic.claude-3-haiku-20240307-v1:0
BEDROCK_REGION       = us-east-1
TABLE_NAME           = ModerationHistory
GSI_NAME             = timestamp-index
CORS_ORIGIN          = <domain Amplify sau khi deploy frontend>
```

![Lambda overview](/images/5-Workshop/5.4/lambda-overview.png)

![Environment variables](/images/5-Workshop/5.4/lambda-env.png)

Execution role gắn policy least-privilege đã nêu ở mục 5.2:

![IAM role policy](/images/5-Workshop/5.4/iam-role-policy.png)

#### 3. Tích hợp Amazon Bedrock làm trọng tài

Khi confidence của model < 0.7, handler gọi **Claude 3 Haiku** qua `bedrock-runtime` với prompt yêu cầu phân loại và trả về JSON:

```python
body = {
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 200,
    "temperature": 0,
    "messages": [{"role": "user", "content": PROMPT_TEMPLATE.format(text=text[:1000])}],
}
resp = client.invoke_model(modelId=BEDROCK_MODEL_ID, body=json.dumps(body))
```

Thiết kế chịu lỗi: nếu Bedrock lỗi (hết quota, chưa có model access...), handler **không làm gãy request** mà ghi warning vào log và trả về kết quả của model chính. Chi phí LLM chỉ phát sinh trên phần nhỏ request khó — pattern "cascade" cân bằng chi phí và độ chính xác.

Test trực tiếp trên console Lambda với các event mẫu:

Câu nhạy cảm nhãn HATE — confidence 0.77 vẫn trên ngưỡng nên không cần gọi Bedrock, nguồn quyết định vẫn là `model`:

![Test câu HATE](/images/5-Workshop/5.4/lambda-test-clean.png)

Câu teencode "vcl thế, làm ăn như hạch" — model phân vân, Bedrock phân xử thành OFFENSIVE kèm lý do, từ "vcl" nằm trong `flaggedTerms`:

![Test câu teencode qua Bedrock](/images/5-Workshop/5.4/lambda-test-offensive.png)

Câu mơ hồ — confidence thấp, Bedrock kết luận CLEAN với giải thích:

![Test câu mơ hồ qua Bedrock](/images/5-Workshop/5.4/lambda-test-bedrock.png)

#### 4. Tạo REST API với API Gateway

1. API Gateway → **Create REST API** → tên `fcaj-api`.
2. Tạo resource `/moderate` → method **POST** → Integration type: **Lambda proxy** → chọn `fcaj-moderate`.
3. Tạo resource `/history` → method **GET** → Lambda proxy → cùng function (handler tự route theo path).
4. Chọn từng resource → **Actions → Enable CORS**.
5. **Actions → Deploy API** → stage `prod` → nhận **Invoke URL**.

![API Gateway resources](/images/5-Workshop/5.4/apigw-resources.png)

![Stage prod](/images/5-Workshop/5.4/apigw-stage.png)

#### 5. Test end-to-end bằng curl

```bash
API=https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/prod

curl -s -X POST $API/moderate -H 'Content-Type: application/json' \
  -d '{"text": "vcl thế, làm ăn như hạch"}' | python3 -m json.tool

curl -s "$API/history?limit=5" | python3 -m json.tool
```

![Test bằng curl](/images/5-Workshop/5.4/curl-test.png)

Kết quả trả về đầy đủ: `label`, `confidence`, `scores` từng nhãn, `source` (model/bedrock), `flaggedTerms`, `latencyMs`. Các bản ghi cũng đã xuất hiện trong DynamoDB:

![Dữ liệu trong DynamoDB](/images/5-Workshop/5.4/dynamodb-items.png)

Backend hoàn chỉnh — sang bước deploy giao diện.
