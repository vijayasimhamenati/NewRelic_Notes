# Section 8: Change Tracking (Correlating Changes with Performance)

_(Quick transcript cleanup: the auto-captions misheard **New Relic** as "Neuralink", **`entityGuid`** as "entity grid", and **Atlassian Bamboo / TeamCity / Drone.io** as "to run IO".)_

---

## 1. Core Concept: What Is Change Tracking?

Production systems don't just break randomly—most outages follow a change. **Change Tracking** records a timestamped marker in New Relic every time something changes in your code or environment, including:

- Application code deployments
- Infrastructure / cloud changes (e.g., changing an AWS EC2 instance type)
- Container updates (e.g., swapping a Docker base image or version)
- OS patches (Windows/Linux server updates)
- Feature flag toggles, operational changes, or business events (e.g., marketing campaigns)

When performance degrades or errors spike, Change Tracking lets you pinpoint **which exact change happened right before the drop** so you can diagnose or roll back fast.

---

## 2. How to Set Up Change Tracking (Choosing the Right Method)

Start inside any service in New Relic → click **Setup now** under **Deployments** to open the Change Tracking setup wizard. New Relic supports multiple integration methods:

| Method                              | Best Used For                                                                    | Instructor's Recommendation                                                                                                                                     |
| ----------------------------------- | -------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **GitHub Actions**                  | Teams already deploying via GitHub workflows/releases.                           | **Recommended** if your repos and CI/CD live on GitHub.                                                                                                         |
| **New Relic CLI**                   | Self-managed deployment environments (VMs, Docker scripts) or mixed CI/CD tools. | **Recommended** for portability—if you later migrate from Jenkins to Bamboo, TeamCity, or Drone.io, your CLI command stays identical without rewriting plugins. |
| **NerdGraph API**                   | Custom pipelines, internal developer portals, or programmatic automation.        | Flexible, but requires writing GraphQL requests directly.                                                                                                       |
| **CI/CD Plugins (Jenkins / JFrog)** | Legacy/existing tool-specific setups.                                            | Avoid for new setups due to CI/CD vendor lock-in.                                                                                                               |

### Method A: GitHub Actions Setup Flow

1. Select **GitHub Actions** in New Relic and generate/select a **User API Key**.
2. Add two **Repository Secrets** in GitHub (**Settings → Secrets and variables → Actions**):

- `NEW_RELIC_API_KEY`: Your New Relic User API key.
- `NEW_RELIC_DEPLOYMENT_ENTITY_GUID`: The target service's Entity GUID.

3. Create a new GitHub Actions workflow file using the template New Relic generates and commit it.
4. Trigger a **GitHub Release** (create a Git tag, title, and description → publish). The workflow runs and pushes a deployment marker to New Relic (allow a few minutes—up to ~15 minutes—for markers to appear in the UI).

### Method B: New Relic CLI Setup Flow (`changeTrackingCreateEvent`)

1. Ensure the **New Relic CLI** is installed and configured with a profile containing your API key (either `default` or a dedicated deployment profile).
2. In the setup wizard, choose **`changeTrackingCreateEvent`** (the modern, recommended mutation) instead of the legacy `createDeployment` option.
3. Configure your event fields:

- **Category / Event Type:** `Deployment`, `Deployment Lifecycle` (e.g., running scripts, tagging files, deleting artifacts), `Feature Flag`, `Business Event`, `Operational Event`, or `Custom`.
- **Deployment Type:** e.g., `Basic`, `Blue-Green`, or `Canary` (also `Rolling`, `Shadow`, or `Other` in docs).
- **Required / Optional Fields:** `version` is required for `Deployment` events; fill in placeholders for optional fields like `description` if you plan to populate them dynamically in CI/CD.

4. Replace the `null` entity ID in the generated command with the service's actual **Entity GUID** (copied from the service's info/metadata panel) and execute it in your terminal or CI/CD script.

---

## 3. Reading Change Markers & Impact Analysis in New Relic

Once recorded, change markers appear as vertical dotted lines across **both APM and OpenTelemetry services**:

- **Cross-Signal Visibility:** Markers automatically overlay onto charts for **Response Time, Throughput, Errors, and Logs** (as well as custom NRQL time-series charts).
- **Hover & Quick Summary:** Hovering over the marker shows the deployment metadata; the **Last Deployment** summary shows immediate before-vs.-after deltas (e.g., response time down 2%).
- **Detailed Impact View:** Clicking the marker dot opens full change details—showing custom deployment attributes, telemetry shifts (e.g., _+35% more logs_), and any **alerts or incidents** triggered after the change.

---

## 4. What the Course Missed (Official New Relic Docs & Production Gaps)

- **You Don't Actually Have to Hardcode Entity GUIDs (`entitySearch`):** The lecture manually copies the Entity GUID to replace `null`. Per New Relic docs, the `changeTrackingCreateEvent` API and CLI support a dynamic **`entitySearch` query** (e.g., matching by service `name`, `domain`, `type`, and `accountId`). That means your CI/CD pipeline can attach change markers automatically without hardcoding or maintaining GUID secrets per environment.
- **High-Value Metadata Fields You Should Always Populate:** Beyond `version` and `description`, official docs support **`commit`** (exact Git commit SHA), **`changelog`** (link or summary of changes), **`deepLink`** (URL back to the CI/CD run or release), **`user`** (who triggered it), **`groupId`** (links coordinated changes across multiple microservices under one release), and arbitrary **`customAttributes`**.
- **Required Field for Feature Flags:** Just as `version` is mandatory for `Deployment` events, **`featureFlagId`** is required whenever you log a `Feature Flag` change event.
- **Two NRDB Event Types Under the Hood:** Modern events created via `changeTrackingCreateEvent` are stored in NRDB as **`ChangeTrackingEvent`**, whereas older markers created via `changeTrackingCreateDeployment` or legacy REST APIs are stored as **`Deployment`**. When querying history via NRQL, you may need to query both if your organization has legacy pipelines.
- **Unified Cross-Account View & Permissions:** Any New Relic user type (even Basic/Core users) can _create_ change events via NerdGraph, but only **Full Platform Users** can view the curated Change Tracking UI. New Relic also provides a unified Change Tracking page to filter changes across _all_ entities and accounts, plus **Webhooks** to notify teams when changes occur.

---

## 5. AI Agent Automation Lens

- **Make "What Changed?" Step #1 of Every Automated RCA:** In real-world production incidents, a recent deployment, config tweak, or feature flag toggle is the #1 root cause. Whenever an alert fires, your AI agent should immediately query **Change Tracking** (`ChangeTrackingEvent` and legacy `Deployment` events) for the alerting entity—**and its upstream/downstream dependencies**—within the 30–60 minutes preceding the alert.
- **Why `commit`, `changelog`, and `deepLink` Supercharge AI Resolution:** If your CI/CD pipeline populates `commit` (Git SHA) and `changelog` in the Change Tracking event, your AI agent doesn't just say _"errors spiked after v1.0.1."_ It can inspect the exact commit diff or release notes, correlate the changed file with the failing stack trace in New Relic Logs, and recommend a targeted **rollback or hotfix** with high confidence.
- **Use `groupId` for Multi-Service Outages:** During coordinated microservice releases, passing a shared **`groupId`** across all deployed services lets your AI agent instantly recognize that 5 simultaneously alerting services were all part of the same rollout—preventing duplicate Jira tickets and pointing directly to a single shared release failure.
- **Catch Non-Code Causes (Feature Flags & Operational Events):** By tracking `Feature Flag`, `Operational Event` (e.g., server reboots, DB maintenance), and `Business Event` (e.g., Black Friday traffic surge) markers, your agent won't waste time hunting for a code bug when a traffic spike or outage was actually caused by a feature flag flip or marketing campaign launch.
