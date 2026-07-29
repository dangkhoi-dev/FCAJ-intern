---
title: "Blog 1"
date: 2026-07-13
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# Building a serverless content-moderation system with AWS Lambda and Amazon Bedrock

**Author & publisher:** Tran Phan Dang Khoi &emsp;|&emsp; **Published on:** the AWS Study Group Facebook group

**Post link:** posted on 28/07/2026 in the [AWS Study Group VN](https://www.facebook.com/groups/awsstudygroupfcj) group — awaiting moderator approval at the time of submission

#### Summary

The post introduces the FCAJ system's overall architecture: the flow from the React UI (Amplify) through API Gateway to a Lambda container running the classifier, the "arbiter" mechanism calling Bedrock Claude Haiku when the model is uncertain (confidence < 0.7), and history storage in DynamoDB. It highlights the cascade pattern — a small model handles most requests while the LLM handles only hard cases — balancing cost and accuracy, plus practical lessons: Lambda container images for AI models, cold starts, Bedrock model-access requests and least-privilege IAM design.

#### Post screenshot

![Blog 1 on AWS Study Group](/images/3-BlogsPosted/blog1.png)
