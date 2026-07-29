---
title: "Worklog Week 11"
date: 2026-07-13
weight: 11
chapter: false
pre: " <b> 1.11. </b> "
---

### Week 11: NLP fine-tuning, ideation & proposal

**Duration:** 20/07/2026 – 26/07/2026

#### Goals

* Choose the capstone problem and prove the model approach is feasible
* Get the proposal approved

#### Work performed

* Surveyed candidate problems and selected **Vietnamese toxic-text moderation** — a real need with a public benchmark dataset (ViHSD) and weak coverage by English-first commercial APIs
* Ran the first fine-tuning experiments on XLM-RoBERTa-base in Google Colab and built a TF-IDF + Logistic Regression baseline for comparison
* Hit the practical limit immediately: no team member had a local GPU, and the free Colab runtime is reclaimed after a few hours — which capped training at 3 epochs and shaped every later result
* Drew the target architecture, estimated cost, defined success criteria (macro-F1 ≥ 0.85) and wrote the proposal
* Attended the FCAJ community session on 25/07 and presented the team's Value Creation & Delivery Canvas

#### Results

* **Output:** an approved proposal with a full architecture diagram, plus a first working model and baseline to measure against
* An honest early read on the compute constraint, which is analysed in full in section 5.3
