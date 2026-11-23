# Section 12: Introduction to NRQL (New Relic Query Language)

## 1. Core Concept & Navigating the Query Console

**NRQL (New Relic Query Language)** is New Relic's SQL-like query language for fetching, filtering, and visualizing telemetry stored in NRDB. Every built-in chart, custom dashboard, NRQL alert condition, and NerdGraph API query runs on NRQL under the hood.

### Fast Ways to Access & Learn NRQL in the UI

- **Reverse-Engineer Any Chart:** Click the **`...` (ellipsis)** on the top-right corner of any graph in New Relic → **View query** to see the exact NRQL powering that visualization.
- **Query Console (Query Builder):** Click **Query your data** at the bottom-left of any screen, or use keyboard shortcuts:
- **`Ctrl + Shift + O`** → Open Query Console
- **`Ctrl + Shift + M`** → Minimize Query Console

- **Notebooks:** Run multiple queries side-by-side inside notebooks, then save or share them with teammates.
- **Data Explorer (No-Code Query Builder):** Browse Events, Metrics, and Logs visually without writing syntax from scratch—select an event (e.g., `Span`), click an attribute (e.g., `category`), inspect its unique values (e.g., `http`), apply filters, and switch visualizations (**Raw JSON, Line, Area, Billboard**) before pinning to a dashboard.

---

## 2. NRQL Basics & Built-In Defaults

Every NRQL query requires only two clauses: **`SELECT`** and **`FROM`** (and you can start with either one).

- **Basic Pattern:** `SELECT <attribute> FROM <DataType>` (e.g., `SELECT duration.ms FROM Span`)
- **Inspect All Attributes:** `SELECT * FROM <DataType>` (e.g., `SELECT * FROM Span`)

### Automatic Defaults to Remember

1. **Default Time Window:** Always queries the **past 60 minutes** (1 hour ago to now) unless you explicitly change it.
2. **Automatic `timestamp`:** Even if you only select a single attribute (like `duration.ms`), NRQL automatically includes the `timestamp` column in raw event results.

---

## 3. The 4 Core Data Types ("Tables") Covered

You don't need to memorize every attribute—focus on memorizing the core data types first:

| Data Type (Table)         | What It Stores                                                                                                                           | Why It Matters                                                                                      |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **`Span`**                | Individual steps inside distributed transactions across microservices (HTTP requests, DB calls, external API calls).                     | Primary table for tracing request flow and latency across services.                                 |
| **`Metric`**              | Dimensional numeric metrics collected from New Relic agents and OpenTelemetry collectors.                                                | Fast trend analysis, resource usage, and SLA tracking.                                              |
| **`Log`**                 | Consolidated log entries shipped from New Relic agents, log forwarders, or OpenTelemetry. _(Only populated if log shipping is enabled.)_ | Exact error messages, warnings, and stack traces for debugging.                                     |
| **`InfrastructureEvent`** | Critical operational changes on hosts and Kubernetes clusters (e.g., service restarts, config changes, K8s warnings/evictions).          | Correlates host/cluster operational changes with sudden spikes in `Span` latency or `Metric` drops. |

---

## 4. What the Course Missed (Official New Relic Docs & Crucial NRQL Rules)

- **Strict Case Sensitivity (Major Gotcha):** The video writes lowercase `span`, `metric`, and `infrastructure event` in spoken examples. Per official New Relic docs, **Data Type names and Attribute names are strictly case-sensitive** (`Span`, `Metric`, `Log`, `InfrastructureEvent`, `duration.ms`)—querying `FROM span` returns empty results. Only NRQL **clauses and functions** (`SELECT`, `FROM`, `count()`) are case-insensitive.
- **Backticks vs. Single Quotes (Clarification):**
- **Backticks (```):** The video wraps standard names like `category` and `Span` in backticks. Per official docs, backticks are **optional** for standard names (alphanumeric, `_`, `:`, and `.` for attributes) and **only required** when custom names contain spaces, hyphens, or special characters (e.g., `Logged-in user`).
- **Single Quotes (`'`):** Always use **single quotes** for string values in filters (e.g., `WHERE category = 'http'`).

- **Use `keyset()` Instead of `SELECT *` to Discover Attributes:** Instead of running `SELECT *` and scrolling through raw rows to see what columns exist, official docs recommend **`SELECT keyset() FROM <DataType>`**—it returns a clean list of every available attribute name in that table.
- **Query String Size Limit:** Every NRQL query string must be **less than 4 KB**.
- **Other Essential Tables You Will Need in Production:** Beyond the 4 tables above, real-world alert troubleshooting heavily uses:
- **APM:** `Transaction`, `TransactionError`
- **Browser:** `PageView`, `AjaxRequest`, `JavaScriptError`
- **Infrastructure & K8s:** `SystemSample`, `ProcessSample`, `K8sPodSample`, `K8sContainerSample`
- **Synthetics:** `SyntheticCheck`, `SyntheticRequest`
- **Changes:** `ChangeTrackingEvent`

---

## 5. AI Agent Automation Lens

- **Use "View Query" as Your Agent Blueprint:** When designing NRQL query templates for your AI agent's tools (e.g., New Relic MCP), don't guess attribute names from scratch. Open the exact New Relic UI chart an L1/L2 engineer normally checks during an outage, click **`...` → View query**, and adapt New Relic's own proven NRQL directly into your agent's playbook.
- **Guard Against Case-Sensitivity & Naming Drift:** LLMs frequently hallucinate casing (`from span` instead of `FROM Span`) or confuse naming conventions across tables (for example, `Span` and `Transaction` often use `entity.guid` / `entityGuid`, while `Log` uses `snake_case` like `pod_name` and `K8sPodSample` uses `camelCase` like `podName`). Giving your agent a compact schema reference—or letting it call `SELECT keyset()` on unfamiliar tables—eliminates silent empty-result failures.
- **Ban Blind `SELECT *` Calls in Production Agents:** Running `SELECT *` dumps dozens of verbose metadata fields per row and quickly saturates an LLM's context window. Configure your agent to always request **only the specific attributes needed for triage** and always pass an explicit time window anchored to the alert timestamp rather than relying on the 60-minute default.
