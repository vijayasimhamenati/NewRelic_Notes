# Section 10: Synthetic Monitoring (Basics — Availability & Page Load Performance)

_(Quick transcript cleanup: the auto-captions misheard **Apdex** as "aptx / app text". Also note an important unit correction below regarding the Apdex target.)_

---

## 1. Core Concept: What Is Synthetic Monitoring?

**Synthetic Monitoring** proactively tests your websites, web flows, and APIs from the **outside in**—without installing any agent or instrumenting the target application. Because it requires zero instrumentation, you can monitor both your own services and **critical third-party websites or APIs** that your business depends on.

**Common Use Cases:**

- Verifying a website or API is up and responding as expected.
- Validating SSL/TLS certificates haven't expired.
- Detecting broken links on web pages.
- Measuring real browser page-load performance.
- Testing critical user journeys (e.g., adding items to a shopping cart) and triggering alerts on failures.

**UI Navigation:** Main menu → **Synthetic monitoring** (if hidden, go to **All capabilities**, search _Synthetic_, and pin it) → **Monitors** → **Create monitor**.

---

## 2. Basic Monitor Types Covered

| UI Tile / Name            | Official Monitor Type                  | How It Works Under the Hood                                                                                                                                         | Best Used For                                                                                                                  |
| ------------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Availability**          | **Ping Monitor** (`SIMPLE`)            | Uses a lightweight Java HTTP client (not a real browser—**does not execute JavaScript**). Defaults to `HTTP HEAD` requests so it won't inflate page-view analytics. | Fast, lightweight uptime checks, SSL checks, and basic endpoint availability.                                                  |
| **Page Load Performance** | **Simple Browser Monitor** (`BROWSER`) | Loads the full page inside a real, virtualized **Chrome or Firefox** browser with optional device/orientation emulation (desktop, mobile landscape, etc.).          | Measuring true page-load speed, third-party asset impact, JS-rendered content, and visual failures (with failure screenshots). |

---

## 3. Key Configuration & Advanced Options

| Option              | Available On          | What It Does & Why It Matters                                                                                                                                                    |
| ------------------- | --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Text Validation** | Ping & Simple Browser | Checks that a specific expected string appears in the response body. Prevents **"soft 200" false passes** (e.g., a broken error page or reverse proxy returning `HTTP 200`).<br> |

<br>_Tradeoff on Ping:_ Forces `HTTP GET` instead of `HTTP HEAD`, which may count as synthetic page views in your analytics. |
| **Apdex Target ($T$)** | Ping & Simple Browser | Sets the tolerable response-time threshold for SLA/Apdex scoring. _(See correction below: measured in **seconds**, default **7 seconds**—not milliseconds.)_ |
| **Verify SSL** | Ping & Simple Browser | Validates the full SSL/TLS certificate chain on every check so you don't need a separate SSL monitor for basic expiration/validity checks. |
| **Bypass HEAD Request** | Ping only | Forces `HTTP GET` instead of the default `HTTP HEAD` even without text validation. |
| **Treat Redirect as Failure** | Ping only | Marks `3xx` redirects (`301`, `302`) as failures rather than following them—enforcing strict `200 OK` responses. |
| **Browser & Device Emulation** | Simple Browser | Choose **Chrome** or **Firefox**, plus device type and screen orientation (e.g., Desktop vs. Mobile Landscape). |
| **Enable Screenshot** | Simple Browser | Captures a visual screenshot whenever a check fails so you can immediately see what the user saw. |
| **Locations & Period** | Both | Controls _where_ checks originate (public global locations matching your users) and _how often_ they run (e.g., every 1 min, 5 min, 15 min, etc.). |

---

## 4. Reading Synthetic Results in New Relic

Once a monitor runs, three views give you the full picture:

1. **Summary:** Shows overall **Success Rate (%)**, failed checks, failed locations, connection/request/response times against your Apdex target, and toggleable aggregations (**Average, Median, and 95th Percentile**).
2. **Results (Individual Checks):** Lists every single monitor execution so you can inspect individual failures, filter by location, and view failure screenshots.
3. **Resources:** Breaks down every asset loaded on the page (HTML, CSS, JS, images, API calls), highlighting **time spent by third-party domains** and the exact **HTTP status code** returned for each resource.

---

## 5. What the Course Missed (Official New Relic Docs & Production Gaps)

- **Unit Correction — Apdex Target Is in Seconds, Not Milliseconds:** The lecture says Apdex target is in milliseconds with a default of `7`. Per official New Relic documentation, Synthetic Apdex $T$ is measured in **seconds**, with a default of **7.0 seconds (7,000 ms)**.
- **Select at Least 3 Locations (Avoid Single-Location False Positives):** The instructor picks only 1 location (Sydney). New Relic's official best practices explicitly recommend selecting **at least 3 locations**—if a single regional ISP or AWS zone has a transient blip, a 1-location monitor triggers a false outage alert, whereas multi-location alerting lets you fire only when 2+ locations fail simultaneously.
- **Ping Monitors Cannot Validate JavaScript-Rendered Text:** Because Ping monitors use a simple HTTP client rather than a browser, they never execute JavaScript. If you run Text Validation against a React/Angular/Vue SPA using a Ping monitor, it only sees the bare initial HTML shell and will fail. Always use **Simple Browser** for JS-rendered pages.
- **Custom Headers Solve the "Fake Page Views" & WAF Problem:** The lecture warns that `HTTP GET` requests can pollute your analytics. Both Ping and Simple Browser monitors support **Custom Headers**—in production, teams attach a custom header (or filter New Relic's user-agent) so analytics tools exclude synthetic traffic and Web Application Firewalls (WAFs) allow synthetic checks through.
- **Private Locations (Behind the Firewall):** Public locations only reach internet-facing URLs. To monitor internal apps, staging environments, or private VPC APIs, New Relic provides **Private Locations** (containerized synthetic minions deployed inside your own network).
- **Monitor Downtimes & 13-Month Retention:** You can schedule **Monitor Downtimes** to pause synthetic checks during planned maintenance windows. New Relic also retains every individual synthetic check run for **13 months** for long-term SLA tracking.

---

## 6. AI Agent Automation Lens

- **Outside-In vs. Inside-Out Correlation:** Synthetic monitoring catches failures that APM and Infrastructure agents _cannot_ see—such as **DNS misconfigurations, expired SSL certificates, CDN/WAF blocks, or broken frontend reverse proxies**. When a synthetic alert fires, your AI agent should immediately compare it against internal APM traffic:
- **Synthetics failing + APM traffic dropped to zero:** Edge/network/DNS/SSL/Load Balancer outage (traffic isn't even reaching your servers).
- **Synthetics failing + APM healthy:** Likely a WAF blocking synthetic probes, a frontend text-validation change after a UI deploy, or a regional routing issue.

- **Location Triangulation to Eliminate False Alarms:** Teach your AI agent to inspect **how many locations failed** before escalating:
- **1 of 3+ locations failed:** Likely a localized ISP or regional network hiccup—downgrade priority or wait for a second consecutive failure.
- **All locations failed simultaneously:** Confirmed global outage—escalate immediately and check SSL status, DNS, and recent Change Tracking deployments.

- **Third-Party Blame Isolation via Resources:** When a Simple Browser monitor alerts on slow page loads, having your agent inspect the monitor's resource breakdown (`SyntheticRequest` telemetry) lets it instantly separate **your own slow backend API** from **a stalled third-party script, font, or CDN**—preventing engineering teams from wasting time debugging healthy internal code.
