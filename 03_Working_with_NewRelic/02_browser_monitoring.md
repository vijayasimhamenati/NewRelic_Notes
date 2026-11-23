# Section 3: Frontend Observability — New Relic Browser Monitoring

## 1. Core Concept: What Browser Monitoring Does

New Relic **Browser Monitoring** is a **Real User Monitoring (RUM)** solution. While APM monitors your backend servers, the Browser agent runs directly inside your end users' web browsers to capture real-world page load speed, JavaScript errors, API (AJAX) calls, console logs, and user interactions across different devices, browsers, operating systems, and geographies.

- **Where it lives in the UI:** Frontend apps appear under **Browser**—never under **APM & Services** (which is strictly for backend APM and OpenTelemetry services).

---

## 2. Setup & Installation Workflow

To instrument a frontend app, navigate to **Integrations & Agents** (formerly _Add data_) → search for **Browser monitoring**.

### Installation Options (Course + Official Docs)

| Method                                         | Best Used For                                                        | Key Requirement (Per New Relic Docs)                                                                                                                            |
| ---------------------------------------------- | -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Copy/Paste JS Snippet** _(covered in video)_ | Simple HTML sites or quick standalone setups.                        | Paste as close to the top of the `<head>` tag as possible—right after position-sensitive `<meta>` tags (`charset`, `X-UA-Compatible`) and before other scripts. |
| **NPM Package** _(covered in video)_           | Modern JS frameworks like React, Angular, or Vue.                    | Import/initialize at the very first line of your app's entry file before other code so it wraps globals early.                                                  |
| **APM Auto-Injection** _(from docs)_           | Server-rendered, non-SPA apps already running a New Relic APM agent. | APM agent automatically injects and updates the browser script whenever restarted.                                                                              |
| **NerdGraph API** _(from docs)_                | Automating instrumentation across dozens of web apps in CI/CD.       | Generates and configures browser snippets programmatically.                                                                                                     |

- **Distributed Tracing Toggle:** During setup, if your frontend talks to backend microservices, turn **Distributed Tracing ON** so user browser clicks are stitched directly to the backend spans that fulfill them.

---

## 3. The 4 Essential Browser Views to Check First

The Browser UI has many menus, so start with these four core views:

| UI View                   | What It Shows                                                                                                                                                                                                                   | Why It Matters                                                                                                                          |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **1. Web Vitals**         | **LCP** (largest text/image render time), **INP** (Interaction to Next Paint — button/click responsiveness), and **CLS** (unwanted visual layout shifts), broken down by **Page URL, Geography, Device Type, Browser, and OS**. | Directly impacts user experience and Google **SEO rankings** on public websites; helps isolate device- or browser-specific regressions. |
| **2. AJAX**               | All frontend API calls (`fetch` / `XHR`)—tracking **errors, response time, and throughput**.                                                                                                                                    | Quickly reveals if the frontend is failing because an internal or third-party backend API is slow or returning errors.                  |
| **3. Diagnose / Summary** | High-level **Golden Signals** combining Web Vitals, throughput, response time, and error rates.                                                                                                                                 | Gives a single-pane health overview before drilling down.                                                                               |
| **4. Logs**               | Browser console logs collected automatically from your JavaScript code.                                                                                                                                                         | Lets you inspect client-side console output remotely without needing the user's browser DevTools.                                       |

---

## 4. What the Course Missed (Official Docs & Production Gaps)

- **The Distributed Tracing CORS Gotcha:** Simply toggling **Distributed Tracing ON** only works automatically when your frontend and backend share the **same origin**. If your frontend calls a backend API on a different domain or port (standard in React + Microservices), traces **will not connect** unless you configure two extra things:

1. In New Relic Browser **Application Settings**, enable **Cross-Origin Resource Sharing (CORS)** and add your backend origin URLs (including `http://` or `https://` and non-default ports).
2. On your **backend API**, allow the trace headers in `Access-Control-Allow-Headers`: specifically `newrelic`, `traceparent`, and `tracestate`.

- **Browser Agent Editions (`Lite`, `Pro`, `Pro+SPA`):** New Relic offers three agent tiers at **no extra billing difference** (the default is **Pro+SPA**, which enables everything). If someone downgrades an app to **Lite** to reduce overhead, it completely **turns off AJAX tracking, JS Errors, Browser Logs, and Session Traces**.
- **Source Maps Are Essential in Production:** Production JavaScript is minified/bundled, making raw error stack traces unreadable. In production, you must **upload Source Maps** to New Relic so JS errors map back to real file names and line numbers.
- **Privacy, Consent & Noise Controls:** In real-world apps, you should configure **Consent Mode** (for GDPR/user cookie consent), **Data Obfuscation** (to strip PII from URLs/logs), and **Domain/Subdomain Filtering** (so local dev/staging traffic doesn't pollute production browser data).
- **Other High-Value Features in Docs:** Beyond the 4 views above, New Relic Browser also supports **Session Replay** (visual playback of a user's session when an error hits), **WebSockets monitoring**, and **Page Resources** (identifying slow third-party scripts/images).

---

## 5. AI Agent Automation Lens

- **Instant Frontend vs. Backend Blame Isolation:** When a user-facing alert fires (e.g., high frontend error rate or checkout failure), your AI agent can triage the root cause in seconds by cross-checking two signals:
- **High `JavaScriptError` + Normal `AjaxRequest`:** Pure frontend bug (e.g., broken UI release or a crash isolated to Safari/iOS).
- **High `AjaxRequest` errors (4xx/5xx or timeouts):** Backend API or third-party dependency failure—the agent should immediately follow the distributed trace to the backend APM service instead of blaming the frontend team.

- **Slicing Blast Radius by User Dimensions:** Unlike backend alerts, frontend issues are often localized. Instruct your agent to facet browser errors and Web Vitals by **Browser type**, **Device type**, **Geography**, and **Page URL**. Catching _"98% of errors are coming from Chrome on Android in India"_ narrows the fix dramatically.
- **Watch for Broken Trace Links (CORS Misconfiguration):** If your agent sees a Browser AJAX error hitting an internal API, but finds **zero linked backend spans**, don't assume the backend wasn't reached—flag that **Browser-to-Backend Distributed Tracing / CORS headers** may be missing, and fall back to correlating by target URL and timestamp.
