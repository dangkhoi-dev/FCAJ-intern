---
title: "Deploy the backend"
date: 2026-07-13
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

This section deploys the entire backend tier: the DynamoDB history table → the Lambda function from the container image → a REST API via API Gateway → Bedrock integration as the arbiter.

#### 1. Create the `ModerationHistory` DynamoDB table

Key design: partition key `requestId`; to query "the N most recent records" efficiently, the team adds a **GSI `timestamp-index`** with a constant partition key `gsi1pk = "HISTORY"` and sort key `gsi1sk = ISO timestamp`. Fetching history therefore costs **a single Query** reading exactly N items (never a full-table Scan — important for both cost and latency).

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

![DynamoDB table](/images/5-Workshop/5.4/dynamodb-table.png)

![GSI timestamp-index](/images/5-Workshop/5.4/dynamodb-gsi.png)

#### 2. Create the Lambda function from the container image

Console → Lambda → **Create function** → **Container image**:

| Setting | Value | Note |
|---|---|---|
| Function name | `fcaj-moderate` | |
| Image URI | `<ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/fcaj-moderation:v1` | image from section 5.3 |
| Architecture | `x86_64` | matches `--platform linux/amd64` |
| Memory | **2048 MB** | Lambda allocates CPU proportionally to memory — more memory = faster inference |
| Timeout | **30 s** | cold start needs ~5–10s to load the model |

{{% notice info %}}
The team initially configured 1024 MB and observed **Max memory used = 1011 MB** — right at the ceiling, risking out-of-memory. After raising it to 2048 MB the system stabilized and latency improved noticeably. A practical lesson in benchmarking several memory sizes instead of accepting the default.
{{% /notice %}}

Environment variables:

```
CONFIDENCE_THRESHOLD = 0.7
ENABLE_BEDROCK       = true
BEDROCK_MODEL_ID     = anthropic.claude-3-haiku-20240307-v1:0
BEDROCK_REGION       = us-east-1
TABLE_NAME           = ModerationHistory
GSI_NAME             = timestamp-index
CORS_ORIGIN          = <Amplify domain after the frontend deploy>
```

![Lambda overview](/images/5-Workshop/5.4/lambda-overview.png)

![Environment variables](/images/5-Workshop/5.4/lambda-env.png)

The execution role carries the least-privilege policy shown in section 5.2:

![IAM role policy](/images/5-Workshop/5.4/iam-role-policy.png)

#### 3. Integrating Amazon Bedrock as the arbiter

When the model's confidence is < 0.7, the handler calls **Claude 3 Haiku** through `bedrock-runtime` with a prompt that requests a classification returned as JSON:

```python
body = {
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 200,
    "temperature": 0,
    "messages": [{"role": "user", "content": PROMPT_TEMPLATE.format(text=text[:1000])}],
}
resp = client.invoke_model(modelId=BEDROCK_MODEL_ID, body=json.dumps(body))
```

Fault-tolerant by design: if Bedrock errors out (quota, missing model access...), the handler **never fails the request** — it logs a warning and returns the primary model's result. LLM cost is only incurred on the small fraction of hard requests — a "cascade" pattern balancing cost and accuracy.

Testing directly in the Lambda console with sample events:

A sensitive HATE-label sentence — confidence 0.77 stays above the threshold, so the source remains `model`:

![HATE sentence test](/images/5-Workshop/5.4/lambda-test-clean.png)

The teencode sentence "vcl thế, làm ăn như hạch" — the model hesitates, Bedrock arbitrates to OFFENSIVE with a reason, and "vcl" appears in `flaggedTerms`:

![Teencode arbitration test](/images/5-Workshop/5.4/lambda-test-offensive.png)

An ambiguous sentence — low confidence, Bedrock concludes CLEAN with an explanation:

![Ambiguous sentence test](/images/5-Workshop/5.4/lambda-test-bedrock.png)

#### 4. Create the REST API with API Gateway

1. API Gateway → **Create REST API** → name `fcaj-api`.
2. Create resource `/moderate` → method **POST** → Integration type: **Lambda proxy** → select `fcaj-moderate`.
3. Create resource `/history` → method **GET** → Lambda proxy → same function (the handler routes by path).
4. Select each resource → **Actions → Enable CORS**.
5. **Actions → Deploy API** → stage `prod` → note the **Invoke URL**.

![API Gateway resources](/images/5-Workshop/5.4/apigw-resources.png)

![Stage prod](/images/5-Workshop/5.4/apigw-stage.png)

#### 5. End-to-end test with curl

```bash
API=https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/prod

curl -s -X POST $API/moderate -H 'Content-Type: application/json' \
  -d '{"text": "vcl thế, làm ăn như hạch"}' | python3 -m json.tool

curl -s "$API/history?limit=5" | python3 -m json.tool
```

![curl test](/images/5-Workshop/5.4/curl-test.png)

The response contains everything the UI needs: `label`, `confidence`, per-label `scores`, `source` (model/bedrock), `flaggedTerms`, `latencyMs`. The records also appear in DynamoDB:

![Data in DynamoDB](/images/5-Workshop/5.4/dynamodb-items.png)

The backend is complete — on to the frontend deployment.
