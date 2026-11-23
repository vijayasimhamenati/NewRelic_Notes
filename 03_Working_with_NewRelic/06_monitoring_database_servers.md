# Section 7: Infrastructure On-Host Integrations (Server Applications & Databases)

_(Quick transcript cleanup: the auto-captions misheard **MSSQL** as "MySQL" in `mssql-server-integration-installer` and "MSSQL instances", and likely misheard **Nginx** or **Redis** as "Linux" in the opening list of server apps.)_

---

## 1. Core Concept: Going Beyond Host OS Monitoring

Infrastructure monitoring isn't limited to OS-level CPU, memory, and disk. Using **On-Host Integrations (OHI)**, New Relic collects deep, engine-specific telemetry from server applications you host yourself—such as **Microsoft SQL Server, MySQL, MongoDB, Redis, Kafka, ActiveMQ, and Nginx**.

- **Where OHI Fits Best:** Self-hosted / self-managed applications running on **on-premises servers**, **cloud VMs (e.g., EC2)**, or **Docker / Kubernetes containers** where you manage and tune the engine.
- **Managed Cloud Services (e.g., AWS RDS):** Because you don't have OS/server access on managed cloud services, standard host-level agent installation doesn't apply the same way (see _What the Course Missed_ below for how managed services are actually monitored).

---

## 2. General Setup Pattern (SQL Server Walkthrough)

While each server application (SQL Server, Kafka, Redis, MongoDB) has its own integration agent and auth commands, the **architectural pattern is identical across all of them**:

### Step 1: Choose the Deployment Mode

| Target Environment                | Recommended Setup Method                                                                                                | Key Notes                                                                                                                                  |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **SQL Server on Windows Host**    | **Guided Install via New Relic CLI** (`newrelic install -n mssql-server-integration-installer`) or manual host install. | The guided CLI installer for SQL Server targets **Windows hosts only**.                                                                    |
| **SQL Server on Linux or Docker** | **Dockerized Integration Agent** (running the agent as a separate container).                                           | Connect the agent container to your DB container via a **user-defined Docker bridge**, **default bridge**, or the server's **IP address**. |

### Step 2: Create a Dedicated Monitoring Login

1. Connect to the target application (e.g., opening a SQL terminal into the SQL Server container).
2. Run the New Relic-provided setup script (substituting your chosen username/password—conventionally `newrelic`).
3. For SQL Server, the script creates a login and loops through non-system databases to grant read/monitoring access. For tools like **Redis** or **Kafka**, the equivalent step is simply creating a monitoring user/credentials with read permissions.

### Step 3: Run & Connect the Integration Agent

- Once the agent container (or host integration) starts with the target IP/hostname and credentials, it polls the server application automatically and ships engine metrics to New Relic.

---

## 3. What You See in New Relic (Engine-Level Telemetry)

Unlike a standard Infrastructure Host dashboard (which only shows OS CPU/RAM/Disk), an On-Host Integration creates a dedicated entity (e.g., **MSSQL instances**) with a specialized DBA/operator dashboard showing engine internals:

- **Active Connections:** Tracks connection counts and potential connection leaks/saturation.
- **SQL Compilations:** Tracks stored procedure and query plan compilations (high compilations indicate plan-cache churn or unparameterized queries).
- **Buffer Cache Metrics:** Shows how efficiently the database is serving data from memory vs. hitting disk.

---

## 4. What the Course Missed (Official New Relic Docs & Production Gaps)

- **Monitoring Managed Databases (AWS RDS, Azure SQL, GCP Cloud SQL):** The video implies you skip database monitoring for managed services like AWS RDS because you can't tune the underlying engine. In production, you still need engine visibility—New Relic handles this in two ways:

1. **Cloud Integrations (API / CloudWatch Metric Streams):** Pulls managed infrastructure metrics (CPU, IOPS, FreeableMemory, connections) directly from AWS/Azure/GCP APIs.
2. **Remote On-Host / Database 360 Monitoring:** You can still point an integration agent (like `nri-mssql`) or an OpenTelemetry collector **remotely** at an RDS endpoint over the network using a read-only database user to capture internal query, lock, and wait metrics.

- **Never Hardcode Plaintext Passwords (Secrets Management):** In production, never leave DB passwords in plaintext config files or use admin accounts like `sa`. Per New Relic docs, grant **least-privilege permissions** (for SQL Server: `VIEW SERVER STATE`, `VIEW ANY DEFINITION`, and `CONNECT`) and use New Relic's **Secrets Management** (pulling credentials dynamically from HashiCorp Vault, CyberArk, AWS KMS, or environment variables).
- **Custom Queries Support:** Beyond out-of-the-box metrics, integrations like `nri-mssql` allow you to configure **Custom SQL Queries** to pull organization-specific metrics (e.g., replication lag, failed SQL Agent jobs, or queue table depth) directly into NRDB.
- **UI Navigation Update:** The video navigates via **All Entities → MSSQL instances**. Since "All Entities" was removed from the left navigation bar, you now access these under **Infrastructure → Third-party services** (or the **Databases** / **Entity Explorer** views).
- **Polling Overhead & Ingest Control:** On-host integrations poll your database/broker on a set interval (e.g., every 15–30 seconds). Polling too aggressively on a heavily loaded database adds load and increases ingest volume—tune polling intervals and enable/disable heavy metric groups (like per-database or lock stats) intentionally.

---

## 5. AI Agent Automation Lens

- **Completing the Full-Stack RCA Picture:** APM tells your AI agent _which_ microservice slowed down on a database call; **On-Host Integrations tell your agent _why_ the database slowed down**:
- **High Active Connections** → Connection pool exhaustion or connection leak in an upstream microservice.
- **Low Buffer Cache Hit Ratio + High Disk I/O** → Memory pressure or full table scans evicting hot pages from RAM.
- **Spiking SQL Compilations** → Parameter sniffing issues or unparameterized ad-hoc queries from a recent code deployment.

- **Distinguish "Database Down" vs. "Integration Broken":** On-host integrations rely on a dedicated monitoring user and network access. If the `newrelic` DB password expires or a firewall rule changes, database metrics will suddenly flatline. Teach your agent a simple sanity check before declaring a P1 outage:
- **DB metrics stopped + APM throwing DB connection errors** → Real database outage.
- **DB metrics stopped + APM transactions succeeding normally** → Monitoring agent/credential failure (check `NrIntegrationError` events instead of waking up the DBA).

- **Tagging Consistency Matters Here:** Because on-host integrations run as separate containers or daemons, make sure they carry matching environment/service tags (`env:production`, `workload:checkout`) so your AI agent can automatically link an alerting microservice to the exact SQL Server, Kafka, or Redis entity backing it.
