# Section 9: Shipping & Querying Application Logs (Python & Logs UI)

_(Quick transcript cleanup: the auto-captions misheard **`level`** as "label" and **colon (`:`)** as "column". Also note several important corrections below where the lecture diverges from official New Relic Python agent behavior.)_

---

## 1. Core Concept: Why Ship Logs to New Relic?

Shipping application logs directly into New Relic solves two problems:

1. **Eliminates Tool Sprawl:** You don't have to maintain a separate logging stack (like ELK / Elasticsearch-Logstash-Kibana) alongside New Relic.
2. **Completes Root Cause Analysis (RCA):** Metrics and traces tell you _where_ and _when_ something broke; logs provide the exact warning messages, error payloads, and stack traces needed to explain _why_.

---

## 2. Viewing & Searching Logs in the New Relic UI

Inside any instrumented application (**APM & Services → [Your App] → Logs**), New Relic displays color-coded log entries by severity:

- **Blue:** `INFO`
- **Yellow:** `WARNING`
- **Red:** `ERROR`

### Searching Logs with Lucene Query Syntax

The Logs UI search bar uses **Lucene-based query syntax** for fast text and attribute filtering:

- **Attribute Match:** Use a colon (`:`) to match an attribute to a value—for example, `level:"WARNING"` (attribute names and values are case-sensitive, except during wildcard searches).
- **Boolean Logic:** Combine multiple filters using `AND`, `OR`, or `-` (negation, e.g., `-healthcheck`).
- **Wildcards & Phrases:** Wrap multi-word phrases or special characters in double quotes (`"hello world"`), and use `*` for wildcards (`status:*500*`).
- **Numeric Comparisons:** Filter numeric log attributes with operators like `>=` or `<=` (e.g., `http_response_time_ms:>=500`).
- **Switching to NRQL:** Click the **NRQL** button next to the search bar to convert or run full NRQL queries against your log data.

---

## 3. Critical Corrections: How Python Log Shipping Actually Works (Per Official Docs)

The lecture describes writing Python logs to a local `log.txt` file and setting `log_to_new_relic = true` and `log_file_name = log.txt` in `newrelic.ini`. **This is inaccurate for the real New Relic Python agent.** Here is how official New Relic documentation actually handles Python logs:

| Approach                                                           | How It Actually Works                                                                                                                                                                                                                  | Official `newrelic.ini` Settings         |
| ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| **1. Automatic In-Agent Log Forwarding _(Default & Recommended)_** | Since Python agent **v7.12+** (and **enabled by default** since **v7.16+**), the APM agent hooks directly into Python's standard `logging` module in memory and forwards logs straight to New Relic—**no local `.txt` file required**. | `application_logging.enabled = true`<br> |

<br>`application_logging.forwarding.enabled = true` _(on by default)_ |
| **2. Local File + External Log Forwarder _(Fluent Bit / Infra Agent)_** | If you _do_ write logs to a local file on disk (e.g., for Docker/host log files), you ship that file using a **log forwarder** (New Relic Infrastructure agent, Fluent Bit, Fluentd, Logstash)—not the APM agent. You enable **local decorating** so the APM agent stamps tracing metadata onto each line. | `application_logging.enabled = true`<br>

<br>`application_logging.forwarding.enabled = false`<br>

<br>`application_logging.local_decorating.enabled = true` |
| **3. Agent Diagnostic Logs _(What `log_file` Actually Does)_** | The `log_file` and `log_level` settings in `newrelic.ini` are **only for the New Relic agent's own internal troubleshooting logs** (e.g., debugging why the agent can't connect to New Relic)—not for shipping your application logs. | `log_file = /tmp/newrelic-python-agent.log`<br>

<br>`log_level = info` |

---

## 4. What the Course Missed (Production & Architecture Gaps)

- **NRDB Stores Logs—Not a Separate Lucene Database:** The lecture states New Relic runs Lucene as a NoSQL database in the background. In reality, New Relic stores logs inside **NRDB** alongside your metrics, events, and traces; the Logs UI simply provides a **Lucene-based search syntax** as a convenient frontend query language.
- **Automatic "Logs in Context" Metadata:** When using in-agent forwarding or local decorating, the Python agent automatically enriches every log line with **five linking attributes**: `entity.guid`, `entity.name`, `hostname`, `trace.id`, and `span.id`. That is what lets you jump straight from a failing Distributed Trace to the exact log lines emitted during that request.
- **Double-Billing Trap:** If you already run an Infrastructure/Fluent Bit log forwarder on a server _and_ upgrade your Python agent (which turns in-agent forwarding on by default), **every log line gets shipped twice—causing double ingest billing**. Always disable one of the two paths.
- **Built-In Rate Limiting (Sampling):** By default, the Python agent forwards up to **10,000 log events per minute** (`application_logging.forwarding.max_samples_stored` in v11+, or `event_harvest_config.harvest_limits.log_event_data` in v10 and below; configurable up to 100,000). If a noisy bug floods your app with 50,000 logs/minute, logs beyond the cap are **sampled/dropped**.
- **Modern Config Formats:** Python agent v10.3.0+ (on Python 3.11+) also supports `.toml` configuration files (`newrelic.toml` or `pyproject.toml`) in addition to traditional `newrelic.ini` files and `NEW_RELIC_*` environment variables.

---

## 5. AI Agent Automation Lens

- **Use NRQL, Not Lucene, Inside Your AI Agent:** Lucene syntax is great for humans typing in the Logs UI, but your AI agent querying New Relic programmatically (via NerdGraph or MCP) should query the `Log` event type using **NRQL**. Note that Lucene and NRQL do not have a 1-to-1 syntax equivalence.
- **Pivot via `trace.id` and `entity.guid` (Never Raw Keyword Search First):** Because **Logs in Context** automatically stamps `entity.guid`, `trace.id`, and `span.id` onto every log, instruct your AI agent to filter logs by the exact failing `entity.guid` or `trace.id` from a broken span—instead of running broad text searches that pull irrelevant noise from other services.
- **Watch Out for Missing Logs During High-Volume Outages:** During a cascading failure or retry storm, an app can easily breach the Python agent's default **10,000 logs/min** cap and start sampling logs. If your agent sees a massive spike in APM `TransactionError` counts but fewer matching `ERROR` logs than expected, it should recognize log sampling is occurring rather than assuming the errors went unlogged.
- **Control Log Level & PII Hygiene:** Shipping `DEBUG` logs in production blows up both New Relic ingest costs and your AI agent's LLM token budget. Keep production apps at `INFO` or `WARNING`, use structured JSON logging where possible, and ensure sensitive fields are obfuscated before your AI agent ingests log payloads into its prompt context.
