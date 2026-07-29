---
title: "Introduction"
date: 2026-07-13
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

#### The problem

Profane and offensive language is pervasive in comments across Vietnamese social platforms. Manual moderation cannot keep up with content velocity, and the commercial APIs the team surveyed are tuned primarily for English, so Vietnamese teencode and character variants (e.g. "đmm", "vkl", "cl") are what they handle least reliably. This workshop tackles that problem with an automated, bilingual (Vietnamese–English) moderation system at near-zero cost.

#### Overall architecture

The system's processing flow:

1. A user enters a sentence on the **demo website** (React, hosted on Amplify Hosting).
2. **API Gateway** receives the `POST /moderate` request and forwards it to Lambda.
3. **Lambda (container image)** runs the fine-tuned XLM-RoBERTa model (ONNX INT8) and returns a label with confidence.
4. If confidence < 0.7, Lambda calls **Amazon Bedrock (Claude Haiku)** for arbitration.
5. The result is written to **DynamoDB** and returned to the UI, with violating words highlighted.
6. **CloudWatch** records logs, metrics and alarms; **S3** stores datasets and model artifacts; **ECR** hosts the Docker image.

![Architecture diagram](/images/5-Workshop/architecture.png)

#### AWS services used and selection rationale

| Service | Role | Why chosen |
|---|---|---|
| Amplify Hosting | Hosts the demo UI | CI/CD from GitHub, built-in HTTPS, generous free tier |
| API Gateway | REST endpoint | Managed, throttling to prevent abuse |
| Lambda (container) | Model inference | Serverless, pay per request, supports images up to 10 GB |
| Amazon Bedrock | Arbitrates hard cases | Use Claude without self-hosting an LLM |
| DynamoDB | Moderation history | On-demand, free tier, millisecond latency |
| S3 | Datasets + model artifacts | Durable, cheap |
| ECR | Docker image registry | Direct integration with Lambda |
| CloudWatch | Logs / metrics / alarms | Monitoring and measurement as required by the project |
| Organizations + IAM Identity Center | Manages 4 member accounts | Least privilege, environment separation |

#### What you achieve after this workshop

+ A public demo website: enter a sentence → get a classification with highlighting in ~1 second.
+ A `/moderate` API that can be integrated into other applications.
+ An understanding of how to combine a self-trained ML model with an LLM on Bedrock in a complete serverless pipeline.
