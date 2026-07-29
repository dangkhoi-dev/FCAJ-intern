---
title: "Worklog Week 6"
date: 2026-07-13
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Week 6: Lab — Networking & VPC

**Duration:** 15/06/2026 – 21/06/2026

#### Goals

* Build a VPC from an empty CIDR block rather than using the default one
* Understand what actually makes a subnet public

#### Work performed

* Designed and created a VPC with a public and a private subnet across two Availability Zones
* Attached an Internet Gateway, wrote the route tables, and confirmed that a subnet is public only because of its route — not because of a checkbox
* Placed an instance in the private subnet and gave it outbound access through a NAT Gateway, then measured the hourly cost of that NAT Gateway and removed it
* Compared Security Groups (stateful) with Network ACLs (stateless) by deliberately breaking return traffic

#### Results

* **Output:** a working two-tier VPC with verified public/private isolation
* Understood the network model well enough to justify the project's decision to run Lambda outside a VPC — avoiding both NAT cost and ENI cold-start penalty
