---
title: "Worklog Week 12"
date: 2026-07-13
weight: 12
chapter: false
pre: " <b> 1.12. </b> "
---

### Week 12: Project execution — build, test, finalise & submit

**Duration:** 27/07/2026 – 31/07/2026

#### Goals

* Ship the Toxic Text Moderation Platform end to end
* Complete the bilingual report, publish the blog posts, clean up and submit

#### Work performed

* Fine-tuned the final XLM-RoBERTa model, exported it to ONNX and applied dynamic INT8 quantisation to fit the Lambda container
* Deployed the backend: Lambda container image + API Gateway + DynamoDB, with the confidence-threshold cascade to Amazon Bedrock
* Built the React front end and deployed it to Amplify Hosting; ran end-to-end and error-case testing across Vietnamese and English inputs
* Enabled CloudWatch logs, metrics and an error alarm; captured the full screenshot evidence set
* Wrote this bilingual report on the FCJ template, published the team blog posts to the AWS Study Group community
* Cross-reviewed within the team, deployed the report site to GitHub Pages, backed up the source code and notebook to GitHub
* Deleted every AWS resource, verified the final Billing figure, and submitted the link through the FCAJ Portal on 31/07

#### Results

* **Output:** a public demo running end to end with full monitoring, a complete bilingual report, published blog posts, and all resources cleaned up at near-zero cost
* Measured results reported honestly, including where the model fell short of the proposal target and why
