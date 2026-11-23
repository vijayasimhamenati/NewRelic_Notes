# Section 2: Entities, Tags, Workloads & New Relic CLI

## 1. Entities: The Core Building Block

In New Relic, **anything that reports observability signals** (metrics, events, logs, or traces) is an **Entity**. Every entity is categorized by a **Domain** (root category) and a **Type** (specific kind of resource), and assigned a globally unique identifier called an **Entity GUID**.

| Example Resource                           | Entity Domain      | Entity Type      | Where It Appears in New Relic               |
| ------------------------------------------ | ------------------ | ---------------- | ------------------------------------------- |
| Linux Server / Personal Machine / K8s Node | `INFRA`            | `HOST`           | Infrastructure → Hosts                      |
| Python API / Microservice                  | `APM`              | `APPLICATION`    | APM & Services                              |
| Frontend Web App                           | `BROWSER`          | `APPLICATION`    | Browser                                     |
| External / OpenTelemetry Service           | `EXT`              | `SERVICE`        | Services - OpenTelemetry                    |
| Databases, K8s Clusters, AI Agents, MCPs   | Respective domains | Respective types | Dedicated capability menus across New Relic |

- **UI Note:** Older tutorials show an **"All Entities"** left-menu item. New Relic removed it from the primary navigation in favor of capability-specific views (APM & Services, Infrastructure, Workloads, etc.).
- **Finding an Entity GUID in the UI:** Open any entity → click **See metadata and tags** → copy the **Entity GUID**.

---

## 2. Tags: Organizing Entities with Metadata

A **Tag** is a `key:value` metadata pair attached to an entity (e.g., `environment:production`, `team:payments`, `region:us-east-1`).

### Two Sources of Tags (Per New Relic Docs)

1. **Automatically Reported Tags (Immutable via UI/CLI):** Applied automatically from New Relic account metadata (`accountId`, `account`), agent configs, host/cloud metadata, or OpenTelemetry resource attributes. You **cannot** edit or delete these via the UI, CLI, or API—you must change them at the source.
2. **Custom Entity Tags (Mutable via UI/CLI/API):** Added manually or via automation at the entity level. If multiple agents report to the same entity, an entity-level tag applies across all of them.

_(Doc nuance: To rename a tag key, you must delete the old tag and create a new one. Also, for APM entities, if you add or remove tags via UI/CLI and want them reflected in NRQL queries on transaction data, you need to restart the service.)_

---

## 3. Workloads: Grouping Entities into Complete Systems

A **Workload** groups related entities (microservices, hosts, databases, dashboards, and even nested workloads) that work together to power a single system, business function, or team. Instead of monitoring 30 isolated entities, a support team owns **one Workload** to track overall system health, alerts, and incidents in one place.

### What New Relic Docs Add: Two Types of Workloads

| Workload Type                                | How Entities Are Discovered                                                                                                | Best Used For                                                                                                                                                   |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Standard Workloads**                       | Manually selected **or** populated via **dynamic tag-based queries** (new entities with matching tags join automatically). | Infrastructure health, team ownership boundaries, and predictable static/tagged stacks.                                                                         |
| **Intelligent Workloads** _(Public Preview)_ | **Automatically discovered** every 5 minutes via distributed traces (**Transaction 360**).                                 | Business-critical transaction flows (e.g., Checkout) with rapidly changing dependencies; includes **Dynamic Flow Maps**, default/custom KPIs, and AI Summaries. |

### Workload Health Status Rollup Rules (Per New Relic Docs)

Workloads don't just group entities—they calculate an **overall health status** (`Operational`, `Degraded`, `Critical`, `Disrupted`) using three methods (in order of precedence):

1. **Static Status:** Manual override (takes highest precedence—useful during planned maintenance).
2. **Automatic Rules:** Rule-based rollups across entity groups:

- _Roll up worst status:_ Workload turns Degraded/Critical as soon as one (or a threshold count of) entity fails.
- _Roll up best status:_ Workload stays Operational as long as at least one redundant entity (e.g., in an auto-scaling group) is healthy.

3. **Alert Policy & Conditions:** Maps specific alert states directly to workload health.

---

## 4. New Relic CLI: Working with Entities & Tags

The `newrelic entity` command group lets you script entity discovery and tagging—useful for CI/CD pipelines and automation.

| Action                      | Command Pattern                                                        | Notes                                                                                           |
| --------------------------- | ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Search by Name**          | `newrelic entity search --name "ar%"`                                  | Supports exact names or SQL `LIKE` wildcards (`%`). Returns matching entities + all their tags. |
| **Search by Type / Domain** | `newrelic entity search --type HOST`                                   | Can filter by `--type`, `--domain` (`APM`, `INFRA`, `BROWSER`), or `--tags`.                    |
| **Get Entity Tags**         | `newrelic entity tags get --guid <ENTITY_GUID>`                        | Lists all tags currently attached to the entity.                                                |
| **Create Tag**              | `newrelic entity tags create --guid <ENTITY_GUID> --tag key:value`     | Example: `--tag computerName:desktop`.                                                          |
| **Delete Tag**              | `newrelic entity tags delete --guid <ENTITY_GUID> --tag key`           | Deletes the entire tag key (you only pass the key, not the value).                              |
| **Delete Specific Values**  | `newrelic entity tags delete-values --guid <ENTITY_GUID> ...`          | Removes specific values from a multi-value tag without deleting the whole key.                  |
| **Replace Tag**             | `newrelic entity tags replace --guid <ENTITY_GUID> --tag key:newvalue` | Overwrites existing values for that tag key.                                                    |

- **Deprecated Command Warning:** `newrelic entity deployment` is **obsolete** for deployment markers. New Relic replaced it with **Change Tracking** (`newrelic changeTracking create`), which records deployments, feature flags, config changes, commit SHAs, and maintenance windows directly in NRDB.

---

## 5. What the Course Missed (Doc-Verified Best Practices & Pitfalls)

- **Don't Use Tags for Deployments / Release Numbers:** The lecture suggests storing `build number`, `deployment date`, or `release number` in entity tags. Per New Relic's documentation, **tags are for stable, low-cardinality organizational metadata** (`team`, `env`, `tier`, `owner`, `repo`)—never high-cardinality or rapidly changing data. Use **Change Tracking** for deployments, versions, and commit hashes so you get an immutable timeline of changes correlated directly against performance regressions.
- **Tag Hygiene Drives Dynamic Workloads:** If your team enforces a strict tagging schema in CI/CD or Infrastructure-as-Code (e.g., `service`, `env`, `owner_team`), **Standard Workloads** stay automatically up to date without humans ever manually adding new microservices or hosts.
- **Workload Status Delays & Alerting:** You can alert directly on **Workload status changes** (instead of alerting on 50 noisy individual hosts). Keep in mind there can be a brief reconciliation lag (a few minutes) if you switch a workload's health configuration method.

---

## 6. AI Agent Automation Lens

- **Tags Are Your Agent's Routing & Ownership Brain:** When an alert fires, your AI agent should immediately inspect the alerting entity's **tags**. Well-structured tags give your agent instant operational context without hardcoding lookup tables:
- `env:production` vs. `env:staging` → Determines urgency and whether automated remediation is allowed or requires Human-in-the-Loop (HITL) approval.
- `owner_team` / `slack_channel` / `jira_project` → Tells the agent exactly where to route tickets or escalations.
- `repo_url` / `runbook_url` → Points the agent directly to the right codebase or Standard Operating Procedure (SOP) for RCA.

- **Workloads Define Blast Radius:** Instead of searching the entire New Relic account during an outage, your agent can check **which Workload(s)** the alerting entity belongs to and inspect the health of sibling entities inside that same workload. If 4 services in the `Checkout` workload are degraded simultaneously, the agent immediately knows it's a systemic/shared-dependency failure rather than an isolated bug.
- **Change Tracking Over Tags for "What Changed?" Checks:** When your agent starts an RCA, querying **Change Tracking events** (deployments, feature flags, config changes) in the 30–60 minutes before an alert is far more reliable than reading mutable entity tags, because Change Tracking preserves exact timestamps, commit SHAs, and changelogs.
