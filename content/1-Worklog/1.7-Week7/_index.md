---
title: "Worklog Week 7"
date: 2026-07-13
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

### Week 7: Lab — Storage & S3

**Duration:** 22/06/2026 – 28/06/2026

#### Goals

* Use S3 as a real distribution and artefact store, not just a file dump
* Practise the access-control patterns the project would need

#### Work performed

* Created buckets with versioning and default encryption; uploaded and retrieved objects through the console, the CLI and a presigned URL
* Wrote bucket policies and compared the result against IAM identity policies to see which one wins in a conflict
* Set lifecycle rules to move objects to Infrequent Access and then expire them, and confirmed the transitions in the console
* Enabled static website hosting on a bucket and served a test page

#### Results

* **Output:** an S3 bucket used from this point on as the team's distribution point for the exported model artefacts (`model.onnx` + `tokenizer.json`)
* Confident with presigned URLs and bucket policies — the mechanism used to share model files across the team without making anything public
