# Section 4: Measuring User Experience — Apdex & Service Levels (SLIs, SLOs & Error Budgets)

## 1. Apdex (Application Performance Index)

Saying a system is "in good shape" is subjective. **Apdex** turns user satisfaction into a single standardized score between **0** (worst—100% of users frustrated) and **1** (best—100% of users satisfied).

### How Apdex Categorizes Requests

You pick a target response-time threshold **$T$** (in seconds/milliseconds) for your application. Every request falls into one of three buckets:

| Bucket         | Condition                                                      | User Experience                  | Weight in Formula |
| -------------- | -------------------------------------------------------------- | -------------------------------- | ----------------- |
| **Satisfied**  | Response time $\le T$                                          | Fast, responsive                 | $1.0$             |
| **Tolerating** | $T < \text{Response time} \le 4T$                              | Noticeable delay, but acceptable | $0.5$             |
| **Frustrated** | Response time $> 4T$ **OR** server-side error (e.g., HTTP 500) | Too slow or broken               | $0$               |

$$\text{Apdex} = \frac{\text{Satisfied} + (\text{Tolerating} / 2)}{\text{Total Requests}}$$

### Why Teams Are Moving Away from Apdex

While Apdex gives a quick high-level pulse check on the APM Summary page, it is largely being superseded across modern SRE teams by **Service Levels (SLIs & SLOs)**, which are industry-standard across observability platforms and map much more cleanly to business commitments.

---

## 2. Service Levels: SLIs, SLOs, Compliance & Error Budgets

Instead of relying on a single composite score like Apdex, New Relic **Service Levels** let you define explicit, user-focused reliability contracts.

### Core Terminology

| Term                              | Definition                                                                                                                      | Example                                                                                                                                                              |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **SLI (Service Level Indicator)** | The underlying measurement defining what counts as a **Valid** request vs. a **Good** (or **Bad**) request.                     | Ratio of successful requests (or requests under 2 ms) to total valid requests.                                                                                       |
| **SLO (Service Level Objective)** | The **target percentage** your SLI must meet over a specific **time window**.                                                   | _99% of valid requests must succeed over a 7-day rolling window._                                                                                                    |
| **Compliance**                    | The actual percentage of time/requests that met your SLO target across rolling windows (**2-hour, 1-day, 7-day, and 28-day**).  | _13% compliance over 2 hours_ means only 13% of requests met the latency threshold.                                                                                  |
| **Error Budget**                  | The inverse of your SLO ($100\% - \text{SLO Target}$)—your allowable room for failure or slowness before breaching commitments. | A 95% SLO gives you a **5% error budget**. If you haven't used any of it, remaining budget = **100%**; if you've burned half of that 5%, remaining budget = **50%**. |

### How SLIs Are Structured in New Relic

Every SLI follows the same two-part structure built on NRDB metrics or events (you can click **Add baseline service level objectives** to auto-generate defaults, or customize them with your own metrics/traces):

1. **Valid Set (Denominator):** The complete pool of eligible operations (e.g., all transactions that recorded a duration—ignoring null/missing values).
2. **Good or Bad Subset (Numerator):** Always a subset of the Valid set, falling into one of two categories:

- **Success (Availability):** Counts failed requests as **Bad** (e.g., error counts) or non-error requests as **Good**.
- **Latency:** Counts requests faster than or equal to a target duration threshold as **Good**.

---

## 3. What the Course Missed (Official New Relic Docs & Production Gaps)

- **Hidden Apdex Rule — Fast Errors Still Count as Frustrated:** The video only mentions response time $> 4T$ for Frustrated requests. Per New Relic documentation, **any request returning a server-side error (such as HTTP 500) is automatically marked Frustrated regardless of how fast it returned**. A service can have a fast average response time and still show a terrible Apdex score.
- **The Apdex Low-Throughput Trap (`< 100 RPM`):** Apdex is designed for high-throughput apps. If your service receives **fewer than 100 requests per minute (RPM)**, the score becomes statistically unstable and Apdex alert conditions can fire unpredictably.
- **Customizing Apdex $T$:** You aren't stuck with one global $T$ value—you can tune Apdex $T$ per application and even set separate thresholds for critical **Key Transactions** (like `/checkout` vs. `/search`).
- **Why SLO Windows Are 7 or 28 Days (Never 30 Days):** New Relic rolling SLO periods strictly use **complete weeks** (1, 7, or 28 days) so every rolling window includes the exact same number of weekends, preventing weekly traffic seasonality from skewing your compliance numbers. New Relic also enforces **one SLO per SLI**.
- **Two Compliance View Modes:**
- **Operational View:** Shows rolling **2-hour, 1-day, 7-day, and 28-day** windows (color-coded **green** = healthy, **yellow** = budget running low, **red** = budget exhausted) for day-to-day incident response.
- **Period-over-Period View:** Compares calendar weeks or months for leadership reviews and retrospectives.

- **Burn Rate Alerting (Critical for Production):** Alerting directly on raw SLO compliance is noisy if your SLI is volatile. New Relic docs strongly recommend **Error Budget Burn Rate Alerts**—alerting when you are burning through your budget _too fast_ to survive the rest of the period (for example, consuming **2% of a 28-day budget in 1 hour** or **5% in 6 hours**).
- **Faceted Service Levels:** You can group a single SLI's compliance and error budget by up to **2 shared low-cardinality attributes** (such as `awsRegion`, `environment`, or `host`) to immediately see whether an SLO drop is global or isolated to one region. _(Note: Faceted metrics only calculate from the moment you enable them—no historical backfill.)_
- **Service Level Maintenance Windows:** You can schedule one-time or recurring (daily/weekly/monthly) **Maintenance Windows** to exclude planned deployments or chaos testing from burning your error budget.

---

## 4. AI Agent Automation Lens

- **Diagnosing an Apdex Degradation Alert (3-Step Split):** If your AI agent is triggered by an Apdex drop, never assume it's a latency problem. Have the agent immediately run a three-way check:

1. **Check Throughput (`RPM`):** If throughput is under 100 RPM, flag the alert as a potential **low-traffic statistical noise** false positive.
2. **Check Server-Side Error Rate:** Because fast 5xx errors count as Frustrated, verify whether the drop is caused by exceptions rather than slowness.
3. **Sort by Dissatisfaction Percentage:** Identify which specific transaction/endpoint is contributing the highest share of user dissatisfaction—a rarely used slow endpoint matters far less than a high-traffic endpoint crossing $T$.

- **Error Budget as Your Agent's Severity & Action Gate:** Error budgets give your AI agent an objective business-impact scale for deciding how aggressively to act:
- **Fast Burn Rate Alert (2-hour window red / high burn rate):** Active customer-facing outage—trigger immediate autonomous RCA, correlate recent deployments, and escalate high-priority tickets.
- **High Remaining Budget (Green) on a minor spike:** Log the RCA findings for the team without waking up on-call engineers.
- **Exhausted Budget (Red):** Flag in the agent's report that risky deployments should be paused until stability is restored.

- **Use Faceted SLIs to Shrink Triage Time:** When an SLO alert fires on a faceted service level, your agent can inspect the failing facet (`awsRegion`, `host`, `environment`) first—instantly narrowing a broad "latency SLO breached" alert down to _"only `ap-south-1` is burning budget"_ before pulling a single log or trace.
