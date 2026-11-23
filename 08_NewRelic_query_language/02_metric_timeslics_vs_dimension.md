# Section 13: The `Metric` Table — Timeslice Data vs. Dimensional Metrics

## 1. Core Concept: How the `Metric` Table Works

All numeric metrics ingested into New Relic land in the **`Metric`** table. Because `Metric` is **schema-less**, its structure and columns change dynamically depending on _what_ you ingest (APM agents, OpenTelemetry SDKs, Infrastructure agents, or Metric API).

Under the hood, the `Metric` table stores two distinct metric paradigms:

| Feature          | Metric Timeslice Data _(Legacy)_                                                                            | Dimensional Metrics _(Modern)_                                                                                            |
| ---------------- | ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **Origin**       | Legacy New Relic APM agents (still reported for backward compatibility).                                    | OpenTelemetry, Infrastructure agents, Metric API, and newer New Relic agents.                                             |
| **Structure**    | Flat: **timestamp + metric name + value** (e.g., _at 10:00, CPU = 10%_). No arbitrary key-value attributes. | Rich: metric value + attached **key-value attributes (dimensions)** (e.g., `datastoreType = 'MySQL'`, `table = 'users'`). |
| **Naming Style** | Single long string separated by **forward slashes (`/`)** (e.g., `Datastore/statement/MySQL/users/select`). | Clean names separated by **dots (`.`)** (e.g., `go.memory.used`, `apm.service.datastore.operation.duration`).             |
| **Query Power**  | Limited; hard to slice without wildcarding segments of the metric name.                                     | High; lets you filter and group (**`FACET`**) from multiple angles using any attached dimension.                          |

### Inspecting Raw Rows in `Metric`

Running `SELECT *` shows the schema-less columns, including internal JSON-like summary blobs (containing min/max/count/total structures). You never need to parse those JSON blobs manually—use NRQL aggregator functions (`average()`, `count()`, `max()`, etc.) instead.

```sql
-- Inspect raw dimensional metric rows for a specific service
SELECT *
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
  AND metricName IS NOT NULL
LIMIT 10

```

---

## 2. Bridging Legacy APM Timeslices to Dimensional Metrics

New Relic is progressively converting legacy APM timeslice metrics into dimensional metrics so you can slice and `FACET` them using modern NRQL:

- **The `apm.*` Naming Rule:** Dimensional metrics prefixed with **`apm.`** (dot-separated) are **converted versions** of legacy APM timeslice metrics.
- **Key Converted APM Metrics to Remember:**
- `apm.service.transaction.duration`
- `apm.service.transaction.error.count`
- `apm.service.error.count`
- `apm.service.transaction.overview`
- `apm.service.datastore.operation.duration`

- **Core Attributes Always Present on Dimensional Metrics:** Every dimensional metric carries **`metricName`** (used to filter the exact metric) and **`entity.guid`** (used to scope to a specific service)—always verify values are **`IS NOT NULL`** when needed.

### Checking Converted Timeslice Metrics (`metricTimesliceName`)

When inspecting a converted `apm.*` metric, the **`metricTimesliceName`** column reveals which legacy slash-separated timeslice names New Relic consolidated into that single dimensional metric:

```sql
-- See which legacy timeslice names map into apm.service.transaction.overview
SELECT uniques(metricTimesliceName)
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
  AND metricName = 'apm.service.transaction.overview'
SINCE 1 hour ago

```

### Querying Pure Dimensional Metrics (e.g., Go OpenTelemetry)

Both queries below return the same result—filtering by `metricName = 'go.memory.used'` proves it is a true dimensional metric (a non-dimensional metric wouldn't have a `metricName` attribute):

```sql
-- Option A: Filter by metric value not being null
SELECT average(go.memory.used)
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
  AND go.memory.used IS NOT NULL
TIMESERIES 2 minutes

```

```sql
-- Option B: Filter explicitly via metricName (string value uses single quotes)
SELECT average(go.memory.used)
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
  AND metricName = 'go.memory.used'
TIMESERIES 2 minutes

```

---

## 3. Core NRQL Clauses & Discovery Workflows

### 1. Discover Available Metrics & Dimensions First

Because available metrics depend entirely on your agent or OpenTelemetry SDK, start in **Data Explorer → Metrics** (filtered by your entity) or **Service → Data → Metric Explorer**—or run discovery queries directly in the Query Console:

```sql
-- List every unique metric name reported by an entity
SELECT uniques(metricName)
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
SINCE 1 hour ago

```

```sql
-- List every available dimension (attribute) attached to a specific metric
SELECT keyset()
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
  AND metricName = 'apm.service.transaction.duration'
SINCE 1 hour ago

```

### 2. Always Scope by `entity.guid`

Different services and OpenTelemetry libraries attach different attributes to the same metric name. Querying a metric across services without filtering by `entity.guid` frequently returns empty or inconsistent results.

- _Syntax note:_ Chaining multiple `WHERE` clauses works identically to using `AND`.

### 3. Slicing Over Time (`TIMESERIES`) & Grouping (`FACET`)

- **`TIMESERIES`:** Slices aggregated results into time buckets (auto-bucketed by default, e.g., 10-minute chunks). Pass an explicit interval like `TIMESERIES 2 minutes` for finer granularity. _(Cannot be combined with `SELECT _`—requires an aggregator function.)\*
- **`FACET` (NRQL's `GROUP BY`):** Splits results into separate lines/groups by one or more compatible dimensions.

```sql
-- Group transaction overview by transactionType (Web, Other/Background, etc.)
SELECT average(apm.service.transaction.overview)
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
FACET transactionType
TIMESERIES 2 minutes

```

```sql
-- Multi-dimensional FACET: break down database latency by DB engine and table
SELECT average(apm.service.datastore.operation.duration)
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
FACET datastoreType, table
SINCE 1 hour ago
TIMESERIES 5 minutes

```

---

## 4. What the Course Missed (Official New Relic Docs & Crucial Corrections)

- **Correction — `FACET` Does _Not_ Have to Come Before `WHERE`:** The lecture claims `FACET` must always be placed before `WHERE`. Per official NRQL syntax, **standard practice is `WHERE` before `FACET**` (though NRQL's parser is flexible about clause order).
- **Querying Unconverted or Custom APM Timeslices (`newrelic.timeslice.value`):** Because timeslice-to-dimensional conversion is still ongoing, many legacy or custom metrics (like JMX connection pools) don't have an `apm.*` equivalent yet. Official docs expose them via **`newrelic.timeslice.value`** filtered by **`metricTimesliceName`**:

```sql
-- Step 1: Check if a legacy timeslice has a dimensional equivalent
-- (If this returns ONLY 'newrelic.timeslice.value', no apm.* equivalent exists yet)
SELECT uniques(metricName)
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
  AND metricTimesliceName LIKE 'Datastore/%'
SINCE 1 hour ago

```

```sql
-- Step 2: Query unconverted legacy timeslices directly
SELECT average(newrelic.timeslice.value)
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
  AND metricTimesliceName LIKE 'Datastore/statement/%'
FACET metricTimesliceName
TIMESERIES 5 minutes

```

- **Extracting Sub-Fields from Summary Metrics (`getField()`):** To pull a specific sub-field (`min`, `max`, `count`, `total`) out of New Relic's internal JSON summary metric without manual parsing, use **`getField()`**:

```sql
-- Pull the worst single transaction duration per minute from a summary metric
SELECT max(getField(apm.service.transaction.duration, max)) AS 'Worst Duration (s)'
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
SINCE 30 minutes ago
TIMESERIES 1 minute

```

- **Hard Rule — No Mixing Timeslice & Dimensional Metrics in One Query:** Per New Relic docs, you **cannot** combine timeslice metrics (`apm.*` or `newrelic.timeslice.value`) with standard dimensional metrics or event data inside a single NRQL query.
- **`apm.service.transaction.overview` Caveat:** Overview metrics bundle multiple timeslices together and are **strictly designed for `average()` only** (other aggregators produce misleading results).
- **Raw Metric 48-Hour Limit:** Querying unaggregated raw metric points (`RAW`) over a window longer than **48 hours** will fail with a query error.

---

## 5. AI Agent Automation Lens

- **Enforce Mandatory `entity.guid` Scoping in MCP Tools:** Make `entity.guid` a required parameter in your AI agent's metric-querying tool to prevent silent empty-result bugs across heterogeneous APM and OpenTelemetry services.
- **Run `uniques(metricName)` Before Guessing Metric Names:** LLMs routinely hallucinate metric names (e.g., guessing `cpu.utilization` when a Go OTel SDK emits `go.memory.used` or `system.cpu.utilization`). Give your agent a lightweight discovery step first.
- **Use Tight `TIMESERIES` Granularity for Incident Triage:** Default auto-bucketing (e.g., 10-minute chunks) smooths out sharp 90-second spikes. Anchor your agent's queries to `1 minute` buckets around the incident window.
- **Always Issue Two Separate Queries When Correlating APM + Runtime/Infra Metrics:** Because NRDB forbids mixing timeslice-backed `apm.*` metrics with OTel/Infra dimensional metrics in one query, split them into parallel tool calls:

```sql
-- Agent Tool Call 1: APM error spike breakdown (timeslice-backed)
SELECT count(apm.service.transaction.error.count)
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
FACET transactionName
SINCE 15 minutes ago
TIMESERIES 1 minute

```

```sql
-- Agent Tool Call 2: Runtime memory check (pure OTel dimensional metric)
SELECT average(go.memory.used)
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
  AND go.memory.used IS NOT NULL
SINCE 15 minutes ago
TIMESERIES 1 minute

```

---

## 6. Practice Drills & Solutions

Open the Query Console (**`Ctrl + Shift + O`**) and test yourself before checking the solutions.

### Drills

1. **Discovery:** List all unique metric names reported by one of your services in the **last 30 minutes**, then list all dimension keys (`keyset()`) on `apm.service.transaction.duration`.
2. **Endpoint Latency:** Chart the **average** `apm.service.transaction.duration` for a single entity over the **last 1 hour**, faceted by `transactionName`, in **5-minute** `TIMESERIES` buckets.
3. **Span Cross-Check:** From the **`Span`** table, chart the **count** of spans over the **last 1 hour** where `category = 'http'`, faceted by `entity.name`, in **2-minute** buckets.

### Solutions

```sql
-- Solution 1A: Discover metrics
SELECT uniques(metricName)
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
SINCE 30 minutes ago

-- Solution 1B: Discover dimensions on transaction duration
SELECT keyset()
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
  AND metricName = 'apm.service.transaction.duration'
SINCE 30 minutes ago

```

```sql
-- Solution 2: Endpoint latency over time
SELECT average(apm.service.transaction.duration)
FROM Metric
WHERE entity.guid = 'YOUR_ENTITY_GUID'
FACET transactionName
SINCE 1 hour ago
TIMESERIES 5 minutes

```

```sql
-- Solution 3: HTTP span throughput by service
SELECT count(*)
FROM Span
WHERE category = 'http'
FACET entity.name
SINCE 1 hour ago
TIMESERIES 2 minutes

```
