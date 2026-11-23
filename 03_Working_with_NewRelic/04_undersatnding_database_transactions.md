# Section 5: Understanding Database Transactions (APM vs. OpenTelemetry)

_(Quick transcript cleanup: the auto-captions misheard **Killercoda** as "killer coda", **`DB:read_route`** as "DB column read route", and **root entry span** as "route entry span".)_

---

## 1. Two Instrumentation Paths in New Relic

Modern microservices often talk to multiple databases (e.g., Postgres and MySQL). How you inspect their database calls depends on how the service is instrumented:

| Category                   | UI Location                  | How Data Is Collected                                                                          | Characteristics                                                                                                            |
| -------------------------- | ---------------------------- | ---------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **APM Services**           | **Services - APM**           | New Relic language agent (e.g., Python agent).                                                 | Translates signals into OpenTelemetry-compatible distributed traces automatically; cleaner UI integration out of the box.  |
| **OpenTelemetry Services** | **Services - OpenTelemetry** | OpenTelemetry SDK (e.g., Go OTel SDK) sending metrics, logs, and traces directly to New Relic. | Often much **chattier** (higher volume of traces), requiring filtering in the UI to isolate specific endpoints or queries. |

---

## 2. The 3 Views for Investigating Database Transactions

You can inspect database behavior at three progressively broader levels:

### Level 1: Databases View (DB-Centric)

Shows **all database operations** regardless of which web request or API endpoint triggered them.

- **Naming Convention:** `<Database Type> <Table DB.Table or> <Operation>`
- Examples: `Postgres catalog.products select`, `MySQL products select`, `Postgres commit`.
- _Driver nuance:_ Whether the database/schema name appears before the table name depends on the specific database driver.

- **What You See:** Query time, throughput, and time consumed by the calling transaction (e.g., `DB:read_route`).
- **Legacy Caveat:** **Slow SQL traces** is a legacy view—don't be alarmed if it shows no data.

### Level 2: Transactions View (Endpoint-Centric)

Steps one level up to show the **web or API transaction** that initiated the database work.

- **What You See:** Top web transactions by time consumed, throughput, CPU usage, and memory usage.
- **Breakdown Table:** Drilling into a transaction splits total latency into individual segments—application functions, middleware, establishing database connections (Postgres/MySQL), and executing SQL queries. Clicking a database segment jumps directly to the Databases view.

### Level 3: Distributed Tracing View (End-to-End Request Flow)

Because New Relic aligns closely with **OpenTelemetry**, a **Transaction** is represented as a **Trace**, and every step inside it (web handler, internal function, DB query) is a **Span**.

- **Grouping Options:**
- **Root Entity:** Groups traces by the originating service.
- **Root Entry Span:** Groups traces by the top-level API/endpoint name (e.g., `read_route`, `list_products`).

- **Visual Trace Map:** Maps the full multi-hop chain across services and datastores (e.g., `Frontend → Recommendations → Product Catalog → Postgres`).
- **Span Icons (Fast Visual Scanning):**
- **Globe icon:** Web / HTTP transaction span.
- **Database / Datastore icon:** Database operation span.
- **Clock icon:** Internal application code execution span.

---

## 3. Critical Gotcha: Where to Find the SQL Statement (`db.statement` vs. `db.query.text`)

This is one of the most common points of confusion when moving between New Relic APM agents and OpenTelemetry SDKs:

| Instrumentation                        | Attribute Name Carrying SQL | Appears in New Relic UI "Query / Performance" Box?                     | How to View the SQL                                                                                                                                           |
| -------------------------------------- | --------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **New Relic APM Agent**                | `db.statement`              | **Yes** (shows summary in UI; click copy icon for the full statement). | Directly in the span's **Performance / Query** section.                                                                                                       |
| **OpenTelemetry SDK (Modern SemConv)** | `db.query.text`             | **No** (the UI query box specifically looks for `db.statement`).       | Open the span's **Attributes** tab, search for `db.query.text`, and copy it—or remap `db.query.text` → `db.statement` in an OpenTelemetry Collector pipeline. |

---

## 4. What the Course Missed (Official New Relic Docs & Production Gaps)

- **SQL Obfuscation Is ON by Default (Security):** In production, you will rarely see literal values (like user emails, IDs, or SSNs) inside `db.statement` or `db.query.text`. By default, New Relic's **Record SQL** setting is **`Obfuscated`**—it strips string literals and numeric values and replaces them with `?` (e.g., `WHERE ssn = ?`). The other options are **`Off`** (no SQL captured) and **`Raw`** (full literal values—blocked if your account enforces **High-Security Mode**).
- **Client-Side vs. Server-Side DB Visibility (Database 360):** APM and OTel database spans only show the **client-side perspective** (how long the app waited for the query). They do _not_ tell you _why_ the database was slow (e.g., missing index, full table scan, InnoDB row-lock contention, or deadlocks). New Relic solves this with **Database 360** (via the New Relic Distribution of OpenTelemetry / **NRDOT**), which scrapes server-side DB telemetry like active blocking sessions, wait events (`io`, `lock`), deadlocks, and rows examined.
- **APM-to-Database Correlation via SQL Commenting:** Modern New Relic agents (e.g., Java 9.1+, .NET 10.54+) can automatically prepend an invisible SQL comment containing the service GUID (`/* nr_service_guid="..." */`) to outgoing queries. This lets Database 360 tie a heavy lock or slow query on the DB server back to the exact microservice that fired it.
- **OpenTelemetry Semantic Convention Evolution:** OpenTelemetry actively migrated several database attribute names (`db.statement` → `db.query.text`, `db.name` → `db.namespace`, `db.mongodb.collection` → `db.collection.name`). Newer New Relic hybrid agents have started reading _both_ legacy and modern attribute names so tracing stays intact across versions.
- **Connection Pool Exhaustion Shows Up Separately:** Notice in the Transactions breakdown that **creating a DB connection** is tracked separately from **executing the SQL query**. In production outages, a slow endpoint is often caused by connection pool wait/creation time rather than a slow SQL query itself.

---

## 5. AI Agent Automation Lens

- **Always Check Both SQL Attributes:** If your AI agent pulls database spans for RCA across a mixed environment (some services on APM agents, some on OpenTelemetry), hardcoding `db.statement` will cause it to come back empty-handed on OTel services. Design your agent's telemetry parser to fall back gracefully: check `db.statement` **and** `db.query.text` (as well as `db.name` / `db.namespace`).
- **Separate "Slow Query" vs. "N+1 Query" vs. "Connection Bottleneck":** When an alert fires for high API latency and the trace points to a datastore, have your agent inspect three distinct patterns before proposing a root cause:

1. **Single Slow Span:** One query taking seconds → likely a missing index, full table scan, or DB lock/blocking session (check Database 360 / DB host saturation).
2. **High Call Count (N+1 Problem):** Hundreds of fast 2 ms `SELECT` spans inside one parent trace → application-level code/ORM bug, not a database outage.
3. **Slow Connection Setup:** Time spent in DB connection creation rather than query execution → connection pool exhaustion or network/DNS latency to the DB host.

- **Respect Obfuscated SQL in RCA:** Because production queries are obfuscated (`WHERE id = ?`), your agent cannot blindly run the captured SQL string against a database to test it. Instead, instruct the agent to identify the **table**, **operation type**, **calling service/endpoint**, and **recent Change Tracking deployments** to pinpoint what introduced the regression.
