---
title: "Worklog Week 9"
date: 2026-07-13
weight: 9
chapter: false
pre: " <b> 1.9. </b> "
---

### Week 9: Lab — Serverless: Lambda & API Gateway

**Duration:** 06/07/2026 – 12/07/2026

#### Goals

* Build and expose a real HTTP endpoint with no servers
* Understand cold start well enough to design around it

#### Work performed

* Wrote Lambda functions in Python, configured environment variables, memory and timeout, and observed how memory allocation changes both duration and cost
* Measured cold start against warm invocation and read the `REPORT` line in CloudWatch Logs to see billed duration and max memory used
* Fronted the function with API Gateway, configured resources, methods, stages and CORS, and called the endpoint with `curl` and from the browser
* Wrote a least-privilege execution role instead of attaching a managed full-access policy

#### Results

* **Output:** a deployed `/moderate` style endpoint returning JSON, with logs and metrics visible in CloudWatch
* This lab became the direct blueprint for the project backend in section 5.4
