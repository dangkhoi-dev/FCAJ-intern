---
title: "Dọn dẹp tài nguyên"
date: 2026-07-13
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

Chúc mừng bạn đã hoàn thành workshop! Bước cuối cùng — và cũng là thói quen quan trọng nhất khi làm việc với cloud — là dọn sạch toàn bộ tài nguyên để không phát sinh chi phí.

{{% notice warning %}}
Chỉ clean-up sau khi đã chụp đủ toàn bộ screenshot cho báo cáo và quay video demo (nếu có). Xóa rồi không khôi phục lại được.
{{% /notice %}}

#### Thứ tự xóa tài nguyên

Xóa theo thứ tự từ ngoài vào trong (frontend → API → compute → data):

**1. Amplify app** — Console → Amplify → app `fcaj-moderation-ui` → App settings → General settings → **Delete app**.

![Xóa Amplify](/images/5-Workshop/5.6/delete-amplify.png)

**2. API Gateway:**

```bash
aws apigateway delete-rest-api --rest-api-id <api-id> --region ap-southeast-1
```

![Xóa API Gateway](/images/5-Workshop/5.6/delete-apigw.png)

**3. Lambda function:**

```bash
aws lambda delete-function --function-name fcaj-moderate --region ap-southeast-1
```

![Xóa Lambda](/images/5-Workshop/5.6/delete-lambda.png)

**4. DynamoDB table** (GSI bị xóa kèm theo bảng):

```bash
aws dynamodb delete-table --table-name ModerationHistory --region ap-southeast-1
```

![Xóa DynamoDB](/images/5-Workshop/5.6/delete-dynamodb.png)

**5. ECR repository** (kèm toàn bộ image):

```bash
aws ecr delete-repository --repository-name fcaj-moderation --force --region ap-southeast-1
```

![Xóa ECR](/images/5-Workshop/5.6/delete-ecr.png)

**6. CloudWatch alarm + log group:**

```bash
aws cloudwatch delete-alarms --alarm-names fcaj-lambda-errors --region ap-southeast-1
aws logs delete-log-group --log-group-name /aws/lambda/fcaj-moderate --region ap-southeast-1
```

**7. S3** — bucket chứa dataset/model artifacts giữ lại đến khi nộp báo cáo xong (phòng cần build lại), sau đó empty bucket rồi xóa.

#### Kiểm tra chi phí cuối cùng

Vào **Billing → Bills** xác nhận tổng chi phí của cả dự án. Nhờ kiến trúc serverless + free tier, chi phí thực tế của nhóm gần như bằng 0:

![Chi phí cuối](/images/5-Workshop/5.6/billing-final.png)

#### Tổng kết workshop

Qua workshop này, chúng ta đã:

+ Fine-tune và đánh giá một model NLP tiếng Việt (XLM-RoBERTa trên ViHSD) với kỹ thuật chống lệch nhãn.
+ Tối ưu model cho môi trường serverless bằng ONNX INT8 và đóng gói vào Lambda container image.
+ Xây dựng pipeline hai tầng model + LLM (Bedrock Claude Haiku) cân bằng chi phí và độ chính xác.
+ Triển khai hạ tầng serverless hoàn chỉnh: API Gateway, DynamoDB (thiết kế GSI Query-first), Amplify Hosting.
+ Áp dụng các thực hành tốt: IAM least privilege, CloudWatch monitoring/alarm, budget alert, và clean-up có kỷ luật.
