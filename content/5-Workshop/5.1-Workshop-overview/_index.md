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

1. The browser loads the React app from **Amplify Hosting** — static assets only.
2. The browser itself sends `POST /moderate` **directly to API Gateway**, on a different origin from the page. Amplify is not a proxy and is out of the path from here on; this cross-origin call is why CORS must be configured.
3. **API Gateway** invokes **Lambda (container image)** through proxy integration.
4. Lambda runs the fine-tuned XLM-RoBERTa model (ONNX INT8), held inside the image, and produces a label with a confidence score.
5. If confidence < 0.7, the request is escalated to **Amazon Bedrock (Claude 3 Haiku)**.
6. Bedrock returns the adjudicated label to Lambda.
7. Either way, Lambda writes the record to **DynamoDB** exactly once.
8. The response returns through API Gateway to the browser, with violating words highlighted.

Alongside the request path: **IAM** supplies the least-privilege execution role, **CloudWatch** collects logs, metrics and alarms, and **CloudTrail** records account-level API activity. At build time only, **S3** distributes the model artefacts and **ECR** holds the container image, which Lambda pulls and caches when the function is created or updated — never per request.

![Architecture diagram](/images/5-Workshop/architecture.png)

#### AWS services used and selection rationale

| Service | Role | Why chosen |
|---|---|---|
| Amplify Hosting | Hosts the demo UI | CI/CD from GitHub, built-in HTTPS, generous free tier |
| API Gateway | REST endpoint | Managed, throttling to prevent abuse |
| Lambda (container) | Model inference | Serverless, pay per request, supports images up to 10 GB |
| Amazon Bedrock | Arbitrates hard cases | Use Claude without self-hosting an LLM |
| DynamoDB | Moderation history | On-demand, free tier, millisecond latency |
| S3 | Datasets + model artefacts (**build time only**) | Durable, cheap; weights are baked into the image, not fetched at runtime |
| ECR | Container image registry | Direct integration with Lambda; image cached at function create/update, not per request |
| CloudWatch | Logs / metrics / alarms | Monitoring and measurement as required by the project |
| CloudTrail | Account-level API audit trail | Every action attributable to a named identity |
| Organizations + IAM Identity Center | Manages 4 member accounts | Least privilege, environment separation |

#### What you achieve after this workshop

+ A public demo website: enter a sentence → get a classification with highlighting in ~1 second.
+ A `/moderate` API that can be integrated into other applications.
+ An understanding of how to combine a self-trained ML model with an LLM on Bedrock in a complete serverless pipeline.
