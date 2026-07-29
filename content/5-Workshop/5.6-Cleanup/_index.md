---
title: "Clean up"
date: 2026-07-13
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

Congratulations on completing the workshop! The final step — and the most important habit when working with the cloud — is cleaning up every resource so no further costs accrue.

{{% notice warning %}}
Only clean up after you have captured every screenshot for the report and recorded the demo video (if any). Deletion cannot be undone.
{{% /notice %}}

#### Deletion order

Delete from the outside in (frontend → API → compute → data):

**1. Amplify app** — Console → Amplify → app `fcaj-moderation-ui` → App settings → General settings → **Delete app**.

![Delete Amplify](/images/5-Workshop/5.6/delete-amplify.png)

**2. API Gateway:**

```bash
aws apigateway delete-rest-api --rest-api-id <api-id> --region ap-southeast-1
```

![Delete API Gateway](/images/5-Workshop/5.6/delete-apigw.png)

**3. Lambda function:**

```bash
aws lambda delete-function --function-name fcaj-moderate --region ap-southeast-1
```

![Delete Lambda](/images/5-Workshop/5.6/delete-lambda.png)

**4. DynamoDB table** (the GSI is deleted with the table):

```bash
aws dynamodb delete-table --table-name ModerationHistory --region ap-southeast-1
```

![Delete DynamoDB](/images/5-Workshop/5.6/delete-dynamodb.png)

**5. ECR repository** (with all images):

```bash
aws ecr delete-repository --repository-name fcaj-moderation --force --region ap-southeast-1
```

![Delete ECR](/images/5-Workshop/5.6/delete-ecr.png)

**6. CloudWatch alarm + log group:**

```bash
aws cloudwatch delete-alarms --alarm-names fcaj-lambda-errors --region ap-southeast-1
aws logs delete-log-group --log-group-name /aws/lambda/fcaj-moderate --region ap-southeast-1
```

**7. S3** — keep the dataset/model-artifacts bucket until the report is submitted (in case a rebuild is needed), then empty and delete it.

#### Final cost check

Open **Billing → Bills** to confirm the project's total spend. Thanks to the serverless architecture + free tier, the team's actual cost is close to zero:

![Final billing](/images/5-Workshop/5.6/billing-final.png)

#### Workshop wrap-up

Through this workshop, we have:

+ Fine-tuned and evaluated a Vietnamese NLP model (XLM-RoBERTa on ViHSD) with imbalance-aware training.
+ Optimized the model for serverless with ONNX INT8 and packaged it into a Lambda container image.
+ Built a two-tier model + LLM pipeline (Bedrock Claude Haiku) balancing cost and accuracy.
+ Deployed a complete serverless stack: API Gateway, DynamoDB (Query-first GSI design), Amplify Hosting.
+ Applied good practices throughout: IAM least privilege, CloudWatch monitoring/alarms, budget alerts, and disciplined clean-up.
