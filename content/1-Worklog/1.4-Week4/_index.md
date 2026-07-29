---
title: "Worklog Week 4"
date: 2026-07-13
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

### Week 4: Lab — Account foundation & governance

**Duration:** 01/06/2026 – 07/06/2026

#### Goals

* Turn the theory of weeks 1–3 into a real, safely configured AWS account
* Produce the governance artefacts the project would be built on

#### Work performed

* Created an AWS Organization and enabled the organizational structure for the team
* Set up IAM Identity Center, created a permission set per member and stopped using the root account entirely; enabled MFA everywhere
* Created an AWS Budget with an email alarm so any unexpected cost would surface within hours
* Requested and enabled model access for Anthropic Claude Haiku in Amazon Bedrock
* Installed and configured the AWS CLI with SSO profiles and verified access with `aws sts get-caller-identity`

#### Results

* **Output:** a governed multi-account setup — Organization, Identity Center permission sets, budget alarm and Bedrock model access, all captured as console screenshots for section 5.2 of this report
* The account is safe to hand to four people without anyone sharing credentials
