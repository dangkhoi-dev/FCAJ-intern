---
title: "Blog 2"
date: 2026-07-13
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

# From React to the user: Amplify Hosting, API Gateway and CloudWatch for a serverless AI application

**Author & publisher:** Le Duc &emsp;|&emsp; **Published on:** the AWS Study Group Facebook group

**Post link:** posted on 28/07/2026 in the [AWS Study Group VN](https://www.facebook.com/groups/awsstudygroupfcj) group — awaiting moderator approval at the time of submission

#### Summary

The post picks up where most AI write-ups stop. Getting a model to run correctly on your own machine is only half the work; the other half is delivery — where the UI is hosted, how it talks to the backend without breaking, and how you can actually *see* what is happening when it does break.

It covers three things in order:

1. **Amplify Hosting.** Building a React (Vite) app straight from the Git repository — one push equals one deploy, no manual `dist/` upload, served over HTTPS on a CDN with no server to patch and no bucket to accidentally make public. Two practical notes: keep the API endpoint in a build-time environment variable (`VITE_API_URL`) rather than hard-coded in source, so changing backend means changing a variable and rebuilding; and use Amplify's per-branch environments to get a staging URL running alongside production.
2. **The CORS mistake everyone makes on day one.** The classic symptom: `curl` works perfectly, the browser fails with a CORS error in the console. CORS is a *browser* mechanism, not a server one — `curl` is not a browser, so it never complains. Getting past it needs two things on the API Gateway side: an `OPTIONS` preflight method returning `Access-Control-Allow-Origin` / `-Headers` / `-Methods`, **and** those same headers repeated in the real `POST` response. With Lambda proxy integration the CORS headers must come from the function's own response — API Gateway will not add them for you.
3. **A zip-deploy trap:** `index.html` must sit at the very root of the archive. Zipping the folder *containing* `dist` adds an extra directory level, and Amplify returns HTTP 404 even though the deploy reported success. `cd` into `dist` first, then zip.

#### Post screenshot

![Blog 2 on AWS Study Group](/images/3-BlogsPosted/blog2.png)
