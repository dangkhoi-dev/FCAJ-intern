---
title: "Proposal"
date: 2026-07-13
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Toxic Text Moderation Platform
## A serverless profanity / toxic-speech detection system on AWS

### 1. Executive Summary
The Toxic Text Moderation Platform is a real-time text moderation system built by a team of four (Duc, Quoc, Khoi, Quan). It detects profane and offensive language in **Vietnamese and English** comments using a text classifier trained by the team, combined with Amazon Bedrock acting as a "judge" for cases where the model is uncertain. The entire infrastructure is serverless (Lambda, API Gateway, DynamoDB, Amplify), costs nearly nothing at idle, and can be tried directly on a public demo website.

### 2. Problem Statement
*Current problem*
Platforms with user-generated content (comments, chat, forums) in Vietnam must manually moderate large volumes of profane and offensive language. Manual moderation is slow, labor-intensive and inconsistent; and the commercial moderation APIs the team surveyed are tuned primarily for English, so Vietnamese teencode, abbreviations and character variants such as "đmm", "vcl" are the cases they handle least reliably.

*Solution*
The team fine-tunes its own multilingual text classification model on the public Vietnamese **ViHSD** dataset, packaged into a Lambda container for fast, low-cost inference. When the model's confidence falls below a threshold, the sentence is escalated to Amazon Bedrock (Claude) for arbitration — balancing speed, cost and accuracy. Results are stored in DynamoDB for statistics and a moderation history view in the UI.

*Benefits*
Reduces manual moderation workload, answers in about a second on a warm container, covers Vietnamese and — thanks to the multilingual backbone — English as well, and the serverless architecture has no fixed cost and can easily be extended into a moderation API for other applications.

### 3. Solution Architecture
Processing flow: the user enters a sentence on the website (Amplify Hosting) → API Gateway receives a POST /moderate request → Lambda (container image with the fine-tuned model) classifies it → if confidence < 0.7, Lambda calls Amazon Bedrock (Claude Haiku) to re-judge → the result (label, confidence, decision source) is written to DynamoDB and returned to the UI, with profane words highlighted.

![Architecture diagram](/images/2-Proposal/architecture.png)
*(Diagram drawn by Duc & Quan on draw.io.)*

*AWS services used*
- **AWS Amplify Hosting**: hosts the React demo UI, CI/CD from GitHub, built-in HTTPS.
- **Amazon API Gateway**: REST API for the /moderate endpoint, throttling to prevent abuse.
- **AWS Lambda (container image)**: runs the fine-tuned XLM-RoBERTa model (ONNX INT8); serverless, pay per request.
- **Amazon Bedrock (Claude Haiku)**: LLM arbitration for hard cases (sarcasm, new slang) without self-hosting an LLM.
- **Amazon DynamoDB**: stores moderation history (requestId, text, label, confidence, timestamp).
- **Amazon S3**: stores datasets and model artifacts.
- **Amazon ECR**: hosts the Lambda Docker image.
- **Amazon CloudWatch**: logs, metrics, alarms (Lambda errors, latency, cost).
- **AWS Organizations + IAM Identity Center**: manages the four members' accounts with least-privilege access.

*Why serverless*: no servers to manage, automatic scaling with traffic, pay-per-use pricing suitable for a student project, and it showcases many AWS services in one realistic use-case (exceeding the minimum requirement of 3 services).

### 4. AI Model and Data
- **Dataset**: **ViHSD** (~33,000 Vietnamese comments labeled CLEAN/OFFENSIVE/HATE) is the dataset the model is actually trained on, keeping the dataset's original train/val/test split. ViCTSD and the Jigsaw Toxic Comment Classification corpus were surveyed as possible extensions but were not used in the final training run — the multilingual backbone already covers English zero-shot.
- **Baseline**: TF-IDF + Logistic Regression as a benchmark.
- **Main model**: fine-tuned **XLM-RoBERTa-base**. The team originally proposed PhoBERT (vi) + DistilBERT (en), but switched to XLM-R because a single model handles multiple languages (zero-shot cross-lingual) and simplifies deployment; exported to ONNX / INT8-quantized to reduce Lambda cold-start.
- **Evaluation**: Accuracy, Precision/Recall, F1-score, Confusion Matrix; target F1 ≥ 0.85 on the test set.

### 5. Timeline (Jul 15 – Jul 31)
- **Jul 15–19 (Tech phase)**: set up AWS Organization and budget alarms (Quan, Duc); collect data and train models (Khoi, Quoc); build infrastructure and backend (Duc, Quan); demo UI and Amplify deployment (Quoc, Khoi); end-to-end testing.
- **Jul 20–31 (Report phase)**: write the bilingual report following the template, produce result charts, write and publish a blog post to the AWS Study Group, cross-review, and clean up resources.

### 6. Budget Estimation
- AWS Lambda: ~$0 (free tier, ~5,000 demo requests).
- API Gateway: ~$0.02.
- DynamoDB (on-demand): ~$0 (free tier).
- S3: ~$0.05 (2–3 GB of data + models).
- ECR: ~$0.10 (one ~2 GB image).
- Amplify Hosting: ~$0.15.
- Amazon Bedrock (Claude Haiku): ~$0.50 — only sentences the model is unsure about (confidence < 0.7) reach Bedrock, a small share of total traffic, so this line is budgeted with headroom.
- **Estimated total: under $1 for the entire demo period.** Model training runs on Google Colab / SageMaker Studio Lab (free), so no GPU cost is incurred.

### 7. Risk Assessment
- **Lambda cold-start with a large image** (medium impact, high probability): quantize the model, increase Lambda memory, consider provisioned concurrency during the demo.
- **Model weakness on new slang/teencode** (medium, medium): Bedrock arbitration layer plus a normalization dictionary applied before inference.
- **Bedrock budget overrun** (low, low): sensible confidence threshold, budget alarms, input length limits.
- **Schedule slip** (medium, medium): two parallel tech tracks (AWS vs AI/UI), with Jul 19 reserved as a buffer day.

### 8. Expected Outcomes
A public demo website where anyone can enter a sentence and get a classification with profanity highlighting in about a second once the Lambda container is warm; a complete bilingual report following the FCJ template; a technical blog post shared on the AWS Study Group; and infrastructure fully reproducible via the step-by-step guide in the Workshop section.
