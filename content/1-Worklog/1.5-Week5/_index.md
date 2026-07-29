---
title: "Worklog Week 5"
date: 2026-07-13
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Week 5: Lab — Compute & EC2

**Duration:** 08/06/2026 – 14/06/2026

#### Goals

* Practise the full lifecycle of an EC2 instance from the console and the CLI
* Measure what the Free Tier actually covers

#### Work performed

* Launched t3.micro instances from both the console and the CLI, compared AMIs, and connected over SSH and Session Manager
* Attached, formatted, mounted, snapshotted and restored an EBS volume; observed what survives a stop/start and what does not
* Configured Security Groups to expose only port 22 to a single IP, then only port 80 to the world, and verified the difference with `curl`
* Built a small Auto Scaling Group behind an Application Load Balancer, then tore it down after measuring cost

#### Results

* **Output:** a running web server reachable through an ALB, plus a documented teardown procedure
* First-hand understanding of why the project later chose Lambda over EC2: no idle cost, no patching, no capacity planning
