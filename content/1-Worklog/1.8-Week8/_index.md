---
title: "Worklog Week 8"
date: 2026-07-13
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

### Week 8: Lab — Databases: RDS & DynamoDB

**Duration:** 29/06/2026 – 05/07/2026

#### Goals

* Compare a relational and a NoSQL managed database on the same workload
* Design the table the project would use to store moderation history

#### Work performed

* Launched an RDS MySQL instance, connected from an EC2 host, restored a small dataset, then tested a snapshot restore and measured the cost of leaving it running
* Created DynamoDB tables and experimented with partition-key design; deliberately built a hot partition to see what it does to throughput
* Created a Global Secondary Index to support a query pattern the base table could not serve
* Compared provisioned versus on-demand billing on a bursty, unpredictable access pattern

#### Results

* **Output:** the `ModerationHistory` table design — partition key `requestId`, GSI `timestamp-index`, on-demand billing — which went straight into the project unchanged
* Concluded that for a bursty demo workload, on-demand DynamoDB costs near zero while an idle RDS instance does not
