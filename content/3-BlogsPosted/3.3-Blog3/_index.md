---
title: "Blog 3"
date: 2026-07-13
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

# Designing a DynamoDB table for an AI logging system: access patterns, GSI and on-demand

**Author & publisher:** Tran Quan &emsp;|&emsp; **Published on:** the AWS Study Group Facebook group

**Post link:** posted on 28/07/2026 in the [AWS Study Group VN](https://www.facebook.com/groups/awsstudygroupfcj) group — awaiting moderator approval at the time of submission

#### Summary

Everyone building an AI system on AWS focuses on the model. The post argues that the part right after it — where the result goes and how you read it back — is deceptively easy to get wrong precisely because it looks trivial.

The scenario: every call to the AI API produces a record containing the input text, the predicted label, the confidence, whether the LLM had to be consulted, and the latency. The UI needs a "History" screen showing the N most recent records.

**Rule number one: design the table from the QUERY, not from the ENTITY.** This is the biggest difference between DynamoDB and SQL. In SQL you model the entity and then query it however you like. In DynamoDB it is the reverse — you enumerate the access patterns first, then design the keys to serve exactly those patterns.

For this workload there are exactly two access patterns:

1. Fetch one record by id.
2. List the N most recent records, newest first.

Pattern 1 is solved by a partition key of `id` (a UUID). Pattern 2 is the interesting one, and the post walks through **the trap of `Scan` followed by sorting in application code** — which reads the whole table on every page load, costs more with every record added, and breaks silently once the result exceeds the 1 MB page limit. The correct answer is a **Global Secondary Index** with a constant partition key and the timestamp as the sort key, queried in descending order with a `Limit`.

It closes on **on-demand versus provisioned capacity**: for a bursty, unpredictable demo workload, on-demand costs effectively nothing when idle, while provisioned capacity bills whether traffic arrives or not.

#### Post screenshot

![Blog 3 on AWS Study Group](/images/3-BlogsPosted/blog3.png)
