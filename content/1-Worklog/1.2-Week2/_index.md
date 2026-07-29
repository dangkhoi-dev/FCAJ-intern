---
title: "Worklog Week 2"
date: 2026-07-13
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Week 2: Course sprint 2/3 — Compute, networking & storage

**Duration:** 18/05/2026 – 24/05/2026

#### Goals

* Complete the second third of the video curriculum
* Understand how compute, network and storage fit together into an architecture

#### Work performed

* Watched ~100 videos on EC2 (instance families, AMI, EBS vs instance store, Elastic IP, key pairs), Auto Scaling and Elastic Load Balancing
* VPC end to end: CIDR planning, public and private subnets, route tables, Internet Gateway, NAT Gateway, Security Group versus Network ACL
* Storage: S3 storage classes and lifecycle rules, bucket policies, versioning, EBS and EFS, and when each is the right choice
* Traced the request path of a typical three-tier web application on paper to check the concepts held together

#### Results

* Able to read an AWS architecture diagram and explain why each component is there
* Understood the cost implications of NAT Gateway, EBS provisioning and S3 storage class choice — knowledge that later kept the project bill under $1
