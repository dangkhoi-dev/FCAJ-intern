---
title: "Worklog Week 10"
date: 2026-07-13
weight: 10
chapter: false
pre: " <b> 1.10. </b> "
---

### Week 10: Lab — Containers (Docker & ECR) and AI on AWS

**Duration:** 13/07/2026 – 19/07/2026

#### Goals

* Package an application as a container image and run it on Lambda
* Evaluate Bedrock and SageMaker against the project requirements

#### Work performed

* Learned the Docker build/tag/push cycle, wrote multi-stage Dockerfiles, and pushed images to a private ECR repository
* Deployed a Lambda function from a container image and tested it locally first with the AWS Lambda Runtime Interface Emulator (RIE)
* Explored SageMaker notebooks and endpoints, and priced a persistent inference endpoint against per-request Lambda
* Called Amazon Bedrock (Claude Haiku) from Python, experimented with prompt design for classification, and measured latency and cost per call

#### Results

* **Output:** a container image in ECR running successfully as a Lambda function, validated locally through RIE
* Decided the architecture: a self-hosted quantised model in the Lambda container for the common path, escalating to Bedrock only on low confidence — cheap by default, accurate when it matters
