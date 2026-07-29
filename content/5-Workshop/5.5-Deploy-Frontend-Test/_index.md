---
title: "Deploy frontend & testing"
date: 2026-07-13
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

This section deploys the React UI to Amplify Hosting, runs the end-to-end test suite (including error cases) and enables CloudWatch monitoring.

#### 1. Build the frontend with the real API URL

The frontend is a React (Vite) app: a chat-style input → API call → color-coded label + confidence + decision source (🤖 Model / ⚖️ Bedrock) + **red highlighting of violating words** + a history tab reading from DynamoDB.

```bash
cd frontend
cp .env.example .env
# Open .env and set the Invoke URL from section 5.4 (NO trailing slash):
# VITE_API_URL=https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/prod
npm install
npm run build      # outputs the dist/ folder
```

#### 2. Deploy to Amplify Hosting

The team uses manual deployment (no Git connection needed):

1. Console → **Amplify** → **Create new app** → **Deploy without Git**.
2. App name `fcaj-moderation-ui` → drag & drop the **`dist` folder** (or a zip of the **contents inside** dist) → **Save and deploy**.
3. After ~1 minute you get a public domain like `https://<branch>.<app-id>.amplifyapp.com`.

{{% notice warning %}}
If you zip, `index.html` must sit at the **root** of the archive. Zipping the `dist` folder from outside adds an extra directory level → Amplify returns **HTTP 404**. The team hit exactly this and fixed it by `cd dist` before zipping `*`.
{{% /notice %}}

![Amplify deployed](/images/5-Workshop/5.5/amplify-deployed.png)

Once the domain exists, go back to Lambda and update `CORS_ORIGIN` to the Amplify domain (tighter than `*`).

#### 3. End-to-end test suite

Run against the public website, covering both success and error paths:

| # | Test case | Expected | Result |
|---|---|---|---|
| 1 | Clean sentence: "Hôm nay thời tiết đẹp quá..." | CLEAN | ✅ |
| 2 | Teencode: "vcl thế, làm ăn như hạch" | OFFENSIVE, "vcl" highlighted, source Bedrock | ✅ |
| 3 | Direct insult: "Thằng này ngu như bò" | OFFENSIVE, "ngu" highlighted | ✅ |
| 4 | Character evasion: "Đồ ng.u, cút đi" | OFFENSIVE/HATE | ✅ |
| 5 | English: "You are so stupid, shut up" | OFFENSIVE (zero-shot) | ✅ |
| 6 | Empty input | **Analyze** stays disabled, request never leaves the browser; a direct API call returns `EMPTY_TEXT` | ✅ |
| 7 | Input > 2,000 characters | Textarea is capped at 2,000 by `maxLength`; a direct API call returns `TEXT_TOO_LONG` | ✅ |
| 8 | History tab | Freshly tested sentences, newest first | ✅ |
| 9 | Ambiguous sentence (confidence < 0.7) | Source ⚖️ Bedrock + reason line | ✅ |

A clean sentence classified correctly:

![UI clean sentence](/images/5-Workshop/5.5/ui-clean.png)

Teencode profanity highlighted in red, labeled OFFENSIVE:

![UI profanity highlight](/images/5-Workshop/5.5/ui-offensive-highlight.png)

When the model is uncertain — Bedrock arbitrates with a reason:

![UI Bedrock arbitration](/images/5-Workshop/5.5/ui-bedrock.png)

Zero-shot on English:

![UI English](/images/5-Workshop/5.5/ui-english.png)

**Error-input handling (validated at the UI layer, no screenshot needed).** Cases 6 and 7 are handled *before* a request is ever sent, so they produce no observable error screen: the **Analyze** button stays disabled while the textarea is empty, meaning an empty submission is physically impossible; and the textarea carries `maxLength={2000}`, so the browser stops accepting keystrokes at exactly 2,000 characters and the limit can never be exceeded from the UI. The character counter under the box turns amber past 1,800 to warn the user in advance. The backend still enforces both rules independently (`EMPTY_TEXT` and `TEXT_TOO_LONG` in `handler.py`) so that a direct API call — bypassing the frontend — is rejected as well; that server-side path is what the RIE local test suite exercises.

Moderation history read from DynamoDB through the GSI:

![UI history](/images/5-Workshop/5.5/ui-history.png)

#### 4. Monitoring with CloudWatch

**Logs** — every request produces a `REPORT` line with duration and actual memory used; warnings (Bedrock failures, DynamoDB failures) are proactively logged by the handler with a `[WARN]` prefix:

![CloudWatch logs](/images/5-Workshop/5.5/cloudwatch-logs.png)

**Metrics** — Lambda Invocations, Duration and Errors after the test run:

![CloudWatch metrics](/images/5-Workshop/5.5/cloudwatch-metrics.png)

**Alarm** — alerting on any Lambda error:

```bash
aws cloudwatch put-metric-alarm --alarm-name fcaj-lambda-errors \
  --namespace AWS/Lambda --metric-name Errors \
  --dimensions Name=FunctionName,Value=fcaj-moderate \
  --statistic Sum --period 300 --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold --region ap-southeast-1
```

![CloudWatch alarm](/images/5-Workshop/5.5/cloudwatch-alarm.png)

The system now runs end-to-end with full monitoring. One last step: cleaning up.
