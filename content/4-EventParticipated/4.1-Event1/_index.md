---
title: "Event 1"
date: 2026-07-13
weight: 1
chapter: false
pre: " <b> 4.1. </b> "
---

### FCAJ / AWS Study Group community sharing session

&emsp;**Date & time:** 06/06/2026

&emsp;**Location:** Community session of the First Cloud AI Journey program (AWS Study Group)

&emsp;**Role:** Attendee

#### Event content

This was a community sharing session with six back-to-back talks, deliberately mixing hands-on technical content with career and teamwork topics. The speakers were a mix of working engineers and students in the program, and each presented something they had actually built or lived through rather than a generic overview.

| # | Speaker | Topic |
|---|---|---|
| 1 | Bảo Huỳnh — Junior Cloud Native Developer, Endava Vietnam | Docker — a containerization technology |
| 2 | Lê Hoàng Gia Đại — HUTECH | Combining AWS WAF with machine learning for cyber-attack detection on AWS |
| 3 | Nguyễn Quốc Bảo | Multiplayer in the cloud: connecting Godot clients with AWS WebSockets |
| 4 | Trương Huy Phước | The art of effective teamwork |
| 5 | Việt Phát — Swinburne | Building GraphRAG applications with Amazon Bedrock and Amazon Neptune |
| 6 | Trần Trung Vinh — System Administrator, Central Retail Group | From IT helpdesk to senior sysadmin |

The Docker talk worked from virtualization to containerization, compared virtual machines against containers, then walked through a Dockerfile line by line and finished with a live demo. The AWS WAF talk argued that rule-based protection alone cannot catch zero-day or hybrid attacks, and demonstrated a network intrusion-detection layer trained with LightGBM on the CSE-CIC-IDS2018 dataset, wired into WAF, Kinesis Data Firehose, Lambda, Security Hub and GuardDuty. The Godot talk built a real-time multiplayer matchmaking flow on API Gateway WebSocket, Lambda and DynamoDB, and closed by comparing that approach with AWS GameLift. The GraphRAG talk showed why plain RAG fails on multi-hop questions and presented two routes — a fully managed one using Bedrock Knowledge Bases with Neptune Analytics, and a custom one using LlamaIndex with Amazon Neptune. The two non-technical talks covered team collaboration tooling and one speaker's path from a helpdesk role to senior system administrator.

#### Lessons and value gained

Two of the talks changed how I worked on our own project the same week. The Docker session's explanation of layer caching — that a changed instruction invalidates every layer after it — explained why my Lambda container image was rebuilding from scratch every time; moving the dependency install above the model copy in the Dockerfile cut my rebuild loop dramatically. The AWS WAF talk was even more directly relevant: its central argument, that signature-based rules break down on inputs they have never seen, is exactly the argument for why our moderation system cannot be a keyword blacklist, and its treatment of class imbalance mapped almost one-to-one onto the imbalanced CLEAN/OFFENSIVE/HATE distribution in ViHSD that I was fighting at the time.

The Godot talk taught me something I had not internalized about serverless: because Lambda is stateless, any state you care about has to be pushed into a store like DynamoDB, and the cost of doing that badly (scanning a growing table on every message) grows with usage. That reframed our own `ModerationHistory` table as a design decision rather than an afterthought. The GraphRAG talk widened my view of Bedrock well beyond the single model invocation our project uses, and the helpdesk-to-sysadmin talk left me with the two habits I have tried to keep since: never test in production, and document what you configured while you still remember why.

![Photo of the 06/06 session](/images/4-EventParticipated/event1-1.jpg)
