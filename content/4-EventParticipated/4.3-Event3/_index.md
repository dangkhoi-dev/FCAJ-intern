---
title: "Event 3"
date: 2026-07-13
weight: 3
chapter: false
pre: " <b> 4.3. </b> "
---

### FCAJ — Agentic AI Build Week: demo & pitch day

&emsp;**Date & time:** 25/07/2026

&emsp;**Location:** AWS Vietnam office

&emsp;**Role:** Attendee

#### Event content

This was the closing day of the FCAJ Agentic AI Build Week — a hackathon in which teams built agentic AI products on AWS and then pitched them in front of a full room. The venue was packed, roughly a hundred people, and the format was straightforward: each team got the stage, presented the problem they had chosen, demoed what they had actually built during the week, and took questions.

![Opening of the Agentic AI Build Week demo day](/images/4-EventParticipated/event3-1.jpg)

*The opening of the demo day at the AWS office.*

Four teams presented:

| Team | Product | Idea in one line |
|---|---|---|
| **3KA** | The Hackathon Journey | A retrospective on building under time pressure, told through the arc of doubt → flow → pride |
| **One Team** | AI-powered conversational ordering | Let customers order inside the chat app they already use, instead of a separate app |
| **Plan V** | Solution Architect native app | An agent that turns requirements into a first-draft architecture, diagrams and IaC |
| **Signal Scout** | Early detection of corporate strategic change | Watch public signals to spot a company's strategic shift before it is announced |

Plan V's product targeted a real bottleneck in the solution-architect workflow: reading a BRD or PRD, drafting an initial architecture, and producing diagrams under deadline pressure. Their agent ingests natural-language and structured requirements, produces a requirements catalogue in minutes, drafts hybrid-cloud-aware architecture options, generates editable draw.io diagrams using the official AWS icon set, emits IaC, and attaches a directional cost estimate alongside the architecture. Signal Scout presented their solution through a value creation and delivery canvas, naming AWS, LangFuse, TinyFish and Apify as key partners and delivering the result through a self-service dashboard. One Team's pitch, titled "Ordering Without Leaving the Chat", led with the trigger and the problem before showing the product at all.

![A team presenting their value creation and delivery canvas](/images/4-EventParticipated/event3-2.jpg)

*Signal Scout presenting their value creation and delivery canvas.*

![One Team presenting "Ordering Without Leaving the Chat"](/images/4-EventParticipated/event3-3.jpg)

*One Team presenting "Ordering Without Leaving the Chat".*

#### Lessons and value gained

The single most useful thing I took away was Signal Scout's cost slide. They broke their architecture down service by service — Bedrock tokens, AgentCore short-term memory, AgentCore runtime, WAF, Amplify Hosting, CloudWatch, Secrets Manager, DynamoDB, Lambda — and gave a minimum, mid and maximum monthly figure for each, so the audience could see exactly which component would dominate the bill as usage grew. Our own proposal estimated "under 1 USD" as a single number. Seeing theirs made me realise that a single number tells you nothing about which service breaks first under load, and it is the format I would use if I costed this project again.

Plan V changed how I think about what "agentic" actually means. Before this, my mental model of using an LLM on AWS was essentially what our project does — one model invocation that classifies a piece of text and returns an answer. Their product chains a whole professional workflow together, from reading a requirements document through to emitting infrastructure code, and it introduced me to Bedrock AgentCore runtime and memory as managed building blocks I had not worked with. One Team's pitch made a product point that applies directly to our own system: they deliberately did not build a new interface, they went to where users already were. Our moderation service is currently a standalone demo page, but because it is API-first, the same reasoning says its real home is inside whatever platform is doing the moderating rather than a page of its own.

The 3KA talk was the one I did not expect to value. It was not a technical presentation at all — it was an honest account of the middle of a build, the stretch where the thing does not work yet and you are not sure it will. Hearing a team say that out loud in front of a hundred people was a useful counterweight to the polished demos, and a fair description of our own week.
