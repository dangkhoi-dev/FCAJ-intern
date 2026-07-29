---
title: "Workshop"
date: 2026-07-13
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Building a serverless profanity-detection system with Lambda, Bedrock and DynamoDB

#### Overview

In this workshop, we build a complete **real-time text moderation system** on AWS: a user enters any sentence (Vietnamese or English) on a website, and the system classifies it as **clean / offensive / hate** and highlights the violating words.

The key architectural idea is a **two-tier design**:
+ **Tier 1 — self-trained model**: an XLM-RoBERTa model fine-tuned by the team on the public Vietnamese hate-speech dataset ViHSD, exported to ONNX INT8 and packaged as a Lambda container image — fast, near-free inference on CPU.
+ **Tier 2 — Amazon Bedrock**: when the model is uncertain (confidence < 0.7), the sentence is escalated to Claude Haiku on Bedrock for arbitration — handling hard cases such as sarcasm, new slang and teencode.

Moderation results are stored in **DynamoDB** and shown as a history view in the React UI hosted on **Amplify Hosting**.

![Architecture diagram](/images/5-Workshop/architecture.png)

#### Content

1. [Workshop overview](5.1-workshop-overview/)
2. [Prerequisite](5.2-prerequiste/)
3. [Train & package the model](5.3-train-package-model/)
4. [Deploy the backend: DynamoDB, Lambda, API Gateway, Bedrock](5.4-deploy-backend/)
5. [Deploy the frontend & end-to-end testing](5.5-deploy-frontend-test/)
6. [Clean up resources](5.6-cleanup/)
