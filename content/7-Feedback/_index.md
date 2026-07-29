---
title: "Sharing and Feedback"
date: 2026-07-13
weight: 7
chapter: false
pre: " <b> 7. </b> "
---

Below are my reflections and suggestions after participating in the First Cloud AI Journey program, in the hope of helping the FCAJ team refine the program for future cohorts.

**1. Learning and working environment**
The program's environment is open and strongly encourages self-learning. What I appreciated most is how each team owns its project end-to-end — from idea and architecture to deployment and cost — like a real product rather than a class exercise.

**2. Mentor / admin support**
Mentors responded quickly and gave direction at the right moments, especially when locking down the architecture. Rather than handing out answers, they often asked questions back so the team discovered issues ourselves — initially uncomfortable, but in hindsight the thing that made me grow fastest.

**3. Fit between content and career direction**
The path from core AWS services to a serverless AI project matches my direction well. Being required to combine many services in one realistic use-case turned isolated knowledge into a connected system.

**4. Learning and skill-development opportunities**
Beyond the technical side, I gained soft skills: dividing and coordinating work in a four-person team through a shared task board, bilingual documentation, technical blogging for the community, and disciplined cloud cost management.

**5. Suggestions for improvement**
I would suggest: (1) an early session on Amazon Bedrock and its per-region model-access constraints — the team spent quite some time figuring this out alone; (2) GPU quota or more detailed SageMaker Studio Lab guidance for AI-topic teams; (3) an updated report template compatible with recent Hugo versions to avoid build errors when producing the report website.

**7. After submission — the feedback and enhancement phase (01/08 – 14/08/2026)**
The report was submitted on 31/07/2026, but the HCMUT external-internship period runs to 14/08/2026. The team used the remaining two weeks to close the loop rather than stop:

* **Collecting feedback.** We gathered comments on the three blog posts from the AWS Study Group community and questions raised during the 25/07 community session, and turned them into a prioritised list of issues.
* **Acting on the model result.** The clearest piece of feedback was on the evaluation in section 5.3 — the fine-tuned model scored below the TF-IDF baseline on macro-F1 because free-tier Colab capped training at 3 epochs. In this phase we re-ran training on a paid GPU runtime with early stopping on macro-F1, and began a comparison against a Vietnamese-pretrained encoder. The results are not included in the graded report, which reports only what was measured by the submission date.
* **Hardening the system.** Reducing cold start by trimming the container image, adding input-length validation on the server side rather than relying on the browser `maxLength`, and adding a per-IP rate limit at API Gateway.
* **Documentation.** Writing a deployment runbook so a new team member can reproduce the whole stack from an empty AWS account.

This phase is what the two weeks after submission were for, and it is where most of the "what we would do differently" items in section 5.3 were actually attempted.

**6. Recommending the program**
I will definitely recommend First Cloud AI Journey to friends in the field — it is one of the few programs where students experience the full cycle of building a real cloud product at near-zero cost.
