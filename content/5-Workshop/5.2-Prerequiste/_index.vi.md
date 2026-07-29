---
title: "Chuẩn bị"
date: 2026-07-13
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

Trước khi bắt đầu triển khai, cần chuẩn bị đầy đủ tài khoản AWS, quyền truy cập và công cụ trên máy local.

#### 1. Tài khoản AWS và Organization

Nhóm sử dụng **AWS Organizations** với 4 member account (mỗi thành viên một account) quản lý qua **IAM Identity Center (SSO)**, bật MFA cho tất cả user. Cách làm này tách biệt môi trường làm việc của từng người và áp dụng nguyên tắc least privilege ngay từ tầng tài khoản.

![AWS Organization](/images/5-Workshop/5.2/organization.png)

![IAM Identity Center](/images/5-Workshop/5.2/identity-center.png)

Để tránh phát sinh chi phí ngoài ý muốn, nhóm đặt **AWS Budget 10 USD** kèm cảnh báo email ngay từ ngày đầu:

![Budget alarm](/images/5-Workshop/5.2/budget-alarm.png)

#### 2. Request quyền truy cập model trên Amazon Bedrock

Bedrock yêu cầu request access cho từng model trước khi gọi API. Vào console **Amazon Bedrock** (region đã hỗ trợ, ví dụ `us-east-1`) → **Model access** → **Manage model access** → tick **Anthropic Claude 3 Haiku** → Submit. Trạng thái chuyển thành **Access granted** trong vài phút.

![Bedrock model access](/images/5-Workshop/5.2/bedrock-access.png)

{{% notice warning %}}
Nếu bỏ qua bước này, Lambda sẽ gặp lỗi `AccessDeniedException` khi gọi Bedrock. Hệ thống được thiết kế để không chết vì lỗi này (tự fallback về kết quả model), nhưng tính năng trọng tài sẽ không hoạt động.
{{% /notice %}}

#### 3. Công cụ trên máy local

| Công cụ | Phiên bản | Dùng để |
|---|---|---|
| AWS CLI v2 | mới nhất | thao tác AWS từ terminal, đã chạy `aws configure sso` |
| Docker Desktop | mới nhất | build Lambda container image, test local với RIE |
| Node.js | ≥ 18 | build frontend React (Vite) |
| Python | ≥ 3.10 | chạy script hỗ trợ |
| Git | mới nhất | quản lý source code |

Kiểm tra nhanh:

```bash
aws --version
docker --version
node --version
```

#### 4. Source code

Toàn bộ source code của hệ thống nằm trong repo `fcaj-moderation` (link trong mục [References](../../8-references/)):

```
fcaj-moderation/
├── model/                     # Lambda container: Dockerfile, handler, scripts
│   ├── Dockerfile
│   ├── src/                   # handler.py, moderation.py, bedrock_judge.py, history.py
│   ├── artifacts/             # model.onnx + tokenizer.json (tải từ S3, không commit)
│   ├── events/                # event mẫu để test local
│   └── scripts/               # test_local.sh, push_ecr.sh, export_onnx.py
└── frontend/                  # React (Vite) UI
```

#### 5. IAM policy cho Lambda execution role

Lambda chỉ được cấp đúng các quyền cần thiết (least privilege): ghi/đọc bảng DynamoDB lịch sử và gọi đúng model Claude Haiku trên Bedrock.

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

(Quyền ghi CloudWatch Logs lấy từ managed policy `AWSLambdaBasicExecutionRole` gắn kèm khi tạo function.)

Chuẩn bị xong, chúng ta bắt đầu bước quan trọng nhất: huấn luyện và đóng gói model.
