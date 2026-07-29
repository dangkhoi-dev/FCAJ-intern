---
title: "Prerequisite"
date: 2026-07-13
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

Before deploying anything, prepare the AWS accounts, access rights and local tooling.

#### 1. AWS accounts and Organization

The team uses **AWS Organizations** with 4 member accounts (one per member) managed through **IAM Identity Center (SSO)**, with MFA enabled for every user. This isolates each member's working environment and applies the least-privilege principle from the account layer up.

![AWS Organization](/images/5-Workshop/5.2/organization.png)

![IAM Identity Center](/images/5-Workshop/5.2/identity-center.png)

To avoid unexpected charges, the team set a **$10 AWS Budget** with an email alert from day one:

![Budget alarm](/images/5-Workshop/5.2/budget-alarm.png)

#### 2. Request model access on Amazon Bedrock

Bedrock requires per-model access requests before its API can be called. Open the **Amazon Bedrock** console (in a supported region, e.g. `us-east-1`) → **Model access** → **Manage model access** → tick **Anthropic Claude 3 Haiku** → Submit. The status changes to **Access granted** within a few minutes.

![Bedrock model access](/images/5-Workshop/5.2/bedrock-access.png)

{{% notice warning %}}
If you skip this step, Lambda will hit `AccessDeniedException` when calling Bedrock. The system is designed not to fail on this (it falls back to the model's own result), but the arbitration feature will not work.
{{% /notice %}}

#### 3. Local tooling

| Tool | Version | Purpose |
|---|---|---|
| AWS CLI v2 | latest | AWS operations from the terminal, configured via `aws configure sso` |
| Docker Desktop | latest | build the Lambda container image, local testing with RIE |
| Node.js | ≥ 18 | build the React (Vite) frontend |
| Python | ≥ 3.10 | helper scripts |
| Git | latest | source control |

Quick check:

```bash
aws --version
docker --version
node --version
```

#### 4. Source code

All source code lives in the `fcaj-moderation` repository (link in [References](../../8-references/)):

```
fcaj-moderation/
├── model/                     # Lambda container: Dockerfile, handler, scripts
│   ├── Dockerfile
│   ├── src/                   # handler.py, moderation.py, bedrock_judge.py, history.py
│   ├── artifacts/             # model.onnx + tokenizer.json (downloaded from S3, not committed)
│   ├── events/                # sample events for local testing
│   └── scripts/               # test_local.sh, push_ecr.sh, export_onnx.py
└── frontend/                  # React (Vite) UI
```

#### 5. IAM policy for the Lambda execution role

Lambda receives only the permissions it needs (least privilege): read/write on the moderation-history DynamoDB table and invoking exactly the Claude Haiku model on Bedrock.

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

(CloudWatch Logs permissions come from the managed policy `AWSLambdaBasicExecutionRole` attached when the function is created.)

With everything prepared, we move to the most important step: training and packaging the model.
