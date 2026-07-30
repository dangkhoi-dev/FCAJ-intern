---
title: "Self-evaluation"
date: 2026-07-13
weight: 6
chapter: false
pre: " <b> 6. </b> "
---

During my internship at the **First Cloud AI Journey (AWS Vietnam)** program from **15/06/2026** to **14/08/2026**, I had the opportunity to apply what I learned to a complete real-world project — the serverless profanity-moderation system **Toxic Text Moderation Platform**.

### My role in the team

I served as **team leader** of the four-member team. Concretely, my responsibilities were:

**1. Training the model.** I owned the entire model track end to end: preparing and cleaning the ViHSD data, building the TF-IDF + Logistic Regression baseline for comparison, fine-tuning XLM-RoBERTa-base with class weighting for the 82/7/11 label imbalance, evaluating on the held-out test split, and exporting the result to ONNX with dynamic INT8 quantisation so it would fit inside a Lambda container image.

**2. Technical direction and the architecture.** I chose the problem and the stack, designed the architecture and drew the final diagram myself in draw.io, with Duc and Quan reviewing it. I made the calls the rest of the work depended on: XLM-RoBERTa rather than PhoBERT (for zero-shot English coverage), a self-hosted quantised model rather than a managed inference endpoint (for cost), Lambda outside any VPC (no private resources, so a VPC would only add NAT cost and cold-start latency), and — most importantly — the decision to keep the fine-tuned model in production despite it scoring below the TF-IDF baseline on macro-F1, because its recall on OFFENSIVE and HATE was materially higher and a moderation system should fail towards catching too much rather than too little.

**3. Integrating the model with Amazon Bedrock.** The cascade is my design and my implementation. Rather than treat the model's low precision as a defect to hide, I built the architecture around it: the Lambda container runs the ONNX model on every request, and only predictions below the 0.7 confidence threshold — exactly the band where the model is unreliable — are escalated to Claude Haiku on Bedrock for a second opinion. I wrote the threshold logic, the Bedrock prompt and the response-merging code, and tuned the threshold against the validation set.

**4. Controlling and testing the architecture flow.** I owned correctness of the end-to-end path — Amplify → API Gateway → Lambda → (model | Bedrock) → DynamoDB → response. I tested the flow locally with the Lambda Runtime Interface Emulator before deployment, wrote the test-case matrix covering clean, offensive, hateful, English and edge-case inputs, verified each hop in CloudWatch Logs, and reviewed the IAM policies to keep the Lambda role at least privilege.

Alongside this I wrote and published the team's blog posts on the AWS Study Group community, set the report outline, and finalised and deployed this bilingual report website.

The other three members handled the areas outside the model track. My work sat at the seams between all of them, which is where most of the integration bugs surfaced.

### The team

| Student ID | Full name | Email | Responsibilities |
|---|---|---|---|
| 2352626 | **Tran Phan Dang Khoi** *(team leader — author of this report)* | khoi.tranphandang@hcmut.edu.vn | Model training and evaluation, technical direction, **architecture design and diagram**, Bedrock integration, end-to-end testing of the architecture flow |
| 2353015 | Tran Ba Minh Quan | quan.tranbaminh12@hcmut.edu.vn | Lambda and API Gateway backend, AWS account foundation |
| 2352265 | Le Tran Minh Duc | duc.letranminh@hcmut.edu.vn | DynamoDB, IAM and ECR, front-end delivery pipeline |
| 2353028 | Nguyen Kien Quoc | quoc.nguyenkien@hcmut.edu.vn | React front end |

### Programme attendance

The programme opened **32 sessions** on the registration system over the internship; I registered for **27 of them**. Because the approval ratio was low relative to demand, the sessions I was actually approved for and attended in person at the AWS Vietnam office were:

| Date | Type |
|---|---|
| 04/06/2026 | Study session |
| 05/06/2026 | Study session |
| 12/06/2026 | Study session |
| 29/06/2026 | Study session |
| 25/07/2026 | Community event + 2 study sessions |

That is **7 sessions across 5 days on site**. I also attended the community sessions on 06/06 and 27/06 documented in section 4. I am recording the registration figure alongside the attendance figure because the gap reflects seat allocation rather than a lack of participation on my part — of the 32 sessions opened I put my name down for 27.

To reflect the internship objectively, I evaluate myself against the following criteria:

| # | Criterion | Description | Good | Fair | Average |
| --- | --- | --- | --- | --- | --- |
| 1 | **Professional knowledge & skills** | Fine-tuning NLP models, ONNX optimization, serverless packaging, fluent use of the project's AWS services | ✅ | ☐ | ☐ |
| 2 | **Ability to learn** | Quickly absorbed new technologies (Bedrock, Lambda containers, quantization) in a short time | ✅ | ☐ | ☐ |
| 3 | **Proactiveness** | Researched solutions independently, proposed switching from PhoBERT to XLM-R, volunteered for hard tasks | ✅ | ☐ | ☐ |
| 4 | **Responsibility** | Completed assigned tasks on the team's deadlines | ✅ | ☐ | ☐ |
| 5 | **Discipline** | Followed the program's schedule and rules | ☐ | ✅ | ☐ |
| 6 | **Growth mindset** | Listened to feedback from mentors and teammates and adjusted | ☐ | ✅ | ☐ |
| 7 | **Communication** | Presented technical content clearly through the report, blog and team discussions | ☐ | ✅ | ☐ |
| 8 | **Teamwork** | Worked closely with Quoc (front end) and Quan & Duc (backend and infrastructure) at the integration points | ✅ | ☐ | ☐ |

The area I most need to improve is time management at the end of the project — some report items landed close to the deadline. If I did it again, I would write documentation in parallel with implementation rather than batching it at the end.
