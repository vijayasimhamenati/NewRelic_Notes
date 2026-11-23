# Section 11: Synthetic Monitoring (Advanced — Private Locations & Monitor Downtime)

_(Quick transcript cleanup: the auto-captions misheard **Podman** as "Padman", **Buy Course** as "by course", and **alerts** as "others" in "alerts are going to be raised".)_

---

## 1. Private Locations: Monitoring from Inside Your Own Network

Public synthetic locations test from New Relic's global infrastructure, which cannot reach internal apps or verify connectivity _from_ your corporate network. **Private Locations** let you run synthetic checks directly from inside your **office network, private data center, or cloud VPC**.

**Two Primary Use Cases:**

1. **External SaaS from Your Network's Perspective:** Verify that office staff can actually reach critical SaaS apps (e.g., Salesforce, Microsoft 365) through your corporate firewall, DNS, and ISP.
2. **Internal / Behind-the-Firewall Apps:** Monitor private intranet portals, staging environments, or internal VPC APIs that are never exposed to the public internet.

### How Private Locations Work (Synthetics Job Manager — SJM)

A Private Location is powered by one or more containerized **Synthetics Job Managers (SJMs)** (`newrelic/synthetics-job-manager`) deployed on infrastructure you control.

- **Direction of Traffic:** You are **not** pinging the private location—the SJM runs inside your network, pulls scheduled jobs from New Relic, sends pings/checks _outward_ to your target URLs, and reports results back to New Relic.
- **Supported Deployment Platforms:**
- **Kubernetes** (via Helm chart—e.g., GKE, EKS, AKS) & **OpenShift**
- **Docker Desktop / Docker Engine** (e.g., on an EC2 VM in a VPC or local machine)
- **Amazon ECS** (by supplying the `PRIVATE_LOCATION_KEY` and SJM image)
- **Podman**

### Setup Flow

1. Go to **Synthetic monitoring → Private locations** _(note: this submenu appears after you create your first monitor)_ → **Create private location**.
2. Name the location (e.g., `my-computer`) and generate the **Private Location Key**.
3. Deploy the SJM container to your chosen platform using the generated command/environment variables.
4. Verify the runtime containers are running (e.g., checking for the `ping` container in Docker).
5. Create or edit any synthetic monitor and select your **Private Location** under **Select locations**—results, success rates, and dashboards look identical to public monitors.

---

## 2. Monitor Downtime: Suppressing Checks During Planned Changes

Scripted browser or step monitors rely on specific DOM elements (like clicking an `<a>` anchor link). If a planned deployment takes a site offline temporarily—or introduces breaking UI changes (like replacing an `<a>` link with a `<button>`) before your team updates the synthetic script—monitors will fail continuously and page on-call engineers for a non-existent outage.

**Monitor Downtime** pauses selected synthetic monitors during a scheduled window so they stop running, stop firing alerts, and **preserve your SLA report metrics**.

### Configuration Options

Go to **Synthetic monitoring → Monitor downtime → Create monitor downtime**:

- **Frequency:** **Once** (one-time deployment/maintenance), **Daily**, **Weekly**, or **Monthly**.
- **Duration:** Choose **All day** (e.g., giving your team a full day to update broken scripts), a specific time window (can span past midnight or more than 24 hours).
- **Target Monitors:** Select specific monitors (no limit on how many) or apply to **All monitors**.
- _(UI tip: If the newly created downtime doesn't appear right away, refresh the page.)_

---

## 3. What the Course Missed (Official New Relic Docs & Production Gaps)

### Private Locations (SJM) Deep-Dive

- **Correction — Private Location Key ≠ Ingest Key (and Cannot Be Rotated):** The video calls it "basically just an ingestion key." Per official docs, it is a dedicated **Private Location Key** stored in a secret store that authenticates the SJM with New Relic's **Horde API endpoint**. **It cannot be rotated**—if compromised, you must create a brand-new private location, migrate your monitors to it, and delete the old one. _(UI update: Keys are now only visible inside the **Edit** screen of a private location, not in list columns or metadata/tags.)_
- **Outbound-Only & Per-Check Sandboxing:** You never need to open inbound firewall ports—the SJM only requires **outbound HTTPS access** to New Relic's regional Horde endpoint (US, EU, or JP) and supports corporate HTTP proxies. Under the hood, the SJM dynamically spins up isolated **per-check runtime containers** (`synthetics-ping-runtime`, `synthetics-node-api-runtime`, `synthetics-node-browser-runtime`) and tears them down after each run.
- **Enterprise Security Features You Should Know:**
- **Verified Script Execution (VSE):** Requires a passphrase (`VSE_PASSPHRASE`) before any scripted monitor can run on your private location—preventing unauthorized users in your New Relic org from executing arbitrary code inside your private VPC.
- **Custom CA & mTLS Certificates:** You can mount internal corporate CA certificates (`.pem`) or mTLS client certificates into the SJM so it validates internal HTTPS apps with TLS intact.
- **Local User-Defined Variables:** Store internal test credentials locally on the SJM (`USER_DEFINED_VARIABLES`) so secrets never have to be saved in New Relic's cloud.

- **High Availability & Monitoring the SJM Itself:** Never run a single SJM container in production. One private location can host **multiple SJMs** for redundancy and concurrency. You should also monitor the SJM's own health (`SyntheticsPrivateLocationStatus` and `SyntheticsPrivateMinion` events) for offline minions, queued jobs, or CPU/memory exhaustion.

### Monitor Downtime Deep-Dive

- **Monitor Downtime vs. Alert Muting Rules (Important Distinction):**
- **Monitor Downtime:** **Stops the monitor from running completely** (no telemetry collected, SLA preserved). Best for planned outages or known broken scripts.
- **Alert Muting Rules:** Keeps the monitor **running and collecting telemetry**, but **silences notifications**. Best when you _want_ to observe whether a deployment broke your app without waking up on-call engineers.

- **Programmatic Control & Entity Relationships:** Monitor downtimes don't have to be clicked manually in the UI—they can be created/deleted automatically via the **NerdGraph API** (ideal for CI/CD pipelines) and are tracked as **Entity Relationships** linked to the suppressed monitors.

---

## 4. AI Agent Automation Lens

- **Separate "Private Location Down" from "Target App Down":** When an alert fires on a private-location synthetic monitor, your AI agent should verify **SJM health** (`SyntheticsPrivateLocationStatus` / `SyntheticsPrivateMinion`) before blaming the target app or SaaS vendor.
- **All monitors on that Private Location failing or queueing up + SJM minion count = 0:** Infrastructure/network/proxy/container issue inside your own VPC or office—route to Infra/NetOps, not the app team.
- **SJM healthy + only 1 target URL failing:** Genuine application or SaaS connectivity failure.

- **Detect "Broken Script" vs. "Real Outage" After Deployments:** Scripted browser monitors frequently fail right after a frontend release due to changed selectors (e.g., `<a>` → `<button>`). Teach your AI agent to cross-check three signals when a scripted monitor fails:

1. Did a **Change Tracking deployment** just land on the frontend?
2. Are **Real User Monitoring (Browser)** traffic and **APM** transactions still succeeding normally?
3. Does the synthetic failure message show an element-not-found / selector timeout (`no such element`) rather than an HTTP 5xx or network timeout?

- If all three match, your agent can classify it as a **stale synthetic script** rather than a P1 customer outage—saving hours of wasted escalation.

- **Auto-Schedule Downtimes in CI/CD:** Instead of relying on engineers to remember UI clicks before a breaking release, automate Monitor Downtime creation (or Alert Muting) via **NerdGraph** as a step in your deployment pipeline—keeping your AI alert-resolution agent free from predictable deployment noise.
