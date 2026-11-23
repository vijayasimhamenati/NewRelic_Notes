# Section 6: Code-Level Performance — Thread Profiling

_(Quick transcript cleanup: the auto-captions misheard **`newrelic.ini`** as "new bikini".)_

---

## 1. Core Concept: Why Profiling Is Needed

APM Transactions and Database views tell you _which_ endpoint or SQL query is slow, but they stop at the boundary of your application code. When performance degrades _inside_ your own logic—such as **infinite loops, thread race conditions, lock contention, or unoptimized functions**—you need **Thread Profiling**.

**How New Relic's Thread Profiler Works Under the Hood:**
It is a low-impact, **sampling-based** profiler safe to run in production. Rather than instrumenting every single line of code, it periodically captures the **stack trace of every running thread every 100 ms** over a set duration and aggregates those snapshots into a **Call Tree** showing where your application spent its time.

---

## 2. Supported Languages & Configuration (With Important Doc Corrections)

The APM Thread Profiler is supported for **four language agents** only: **.NET, Java, Python, and Ruby**.

| Language   | Config Key (Per Official Docs)   | Enabled by Default?                | Notes                                                                                           |
| ---------- | -------------------------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------- |
| **.NET**   | N/A (Uses the .NET CLR Profiler) | **Always On** (cannot be disabled) | Supports Windows (.NET Framework & .NET Core 2.0+) and Linux (.NET Core 3.0+ with agent 8.23+). |
| **Java**   | `thread_profiler.enabled`        | **Yes (`true`)**                   | Controlled in `newrelic.yml` or agent config.                                                   |
| **Python** | `thread_profiler.enabled`        | **Yes (`true`)**                   | Controlled in `newrelic.ini`.                                                                   |
| **Ruby**   | `thread_profiler.enabled`        | **Yes (`true`)**                   | Controlled in `newrelic.yml`.                                                                   |

> **Critical Correction to the Video:** The lecture states you must manually add `profile trace = true` because profiling is off by default. Per official New Relic documentation, the actual setting name is **`thread_profiler.enabled`**, and it is **enabled (`true`) by default** across Java, Python, and Ruby—you only need to configure it if it was explicitly disabled.

---

## 3. Running & Reading the Thread Profiler

### How to Run a Session

1. Navigate to **APM & Services → [Select App] → Events → Thread profiler**.
2. Select the **duration** (default is **5 minutes**; minimum is **2 minutes**—longer windows catch more intermittent issues).
3. Select the specific **host/instance** running your code.
4. Click **Start profiler** and wait for the session to finish.

### Reading the Call Tree Results

- Percentages represent the **proportion of sampled thread backtraces** in which a specific call path appeared during the session.
- **Color-Coding (Per Docs):**
- **Red:** Greater than **30%** of samples (primary hotspot).
- **Yellow:** Greater than **10%** of samples.
- **Black:** Less than **10%** of samples.

- **Exporting:** You can export the entire call tree as **JSON** for external analysis or downstream tooling.

---

## 4. What the Course Missed (Official New Relic Docs & Production Gaps)

- **Sleeping & I/O-Blocked Threads Appear in the Call Tree:** A high percentage in the Thread Profiler does **not** automatically mean heavy CPU usage. Per New Relic docs, threads that are **sleeping or blocked waiting on I/O** (network, database, or lock waits) also appear in the sampled call tree. Always cross-check host CPU utilization to distinguish CPU-bound loops from I/O waits.
- **Language-Specific Blind Spots (Per Docs):**
- **.NET:** Only captures **managed threads** (unmanaged/native calls appear generically as `Native:Function Call`) and **does not include line numbers** in the call tree.
- **Java:** Places all threads into the **Other** category rather than splitting Web Request vs. Background threads.
- **Python:** If your app uses **Greenlets** (e.g., `gevent`), the sampler only captures them when they yield/block—not while executing arbitrary code.
- **Ruby:** Requires CRuby 1.9.2+ (`Thread#backtrace`) and does not profile **Resque** background worker processes (only the parent process).

- **The Biggest Operational Limit — On-Demand vs. Continuous Profiling:** The classic Thread Profiler is **reactive and manual**. If a CPU spike lasted 45 seconds at 3:00 AM, clicking "Start Profiler" at 9:00 AM won't catch it. To solve this, modern observability is shifting toward **Continuous Profiling** (always-on low-overhead CPU/Heap sampling—which New Relic has started rolling out in newer agents like Node.js, alongside OpenTelemetry/eBPF continuous profiling) and **Code-Level Metrics** (linking live APM metrics directly to methods in your IDE).

---

## 5. AI Agent Automation Lens

- **Know When Thread Profiling Is (and Isn't) Useful for Automated RCA:** Because the classic APM Thread Profiler is **host-specific, manual, and requires waiting 2–5 minutes**, it is _not_ suited for diagnosing brief, self-healing alert spikes. However, for **sustained** CPU exhaustion or prolonged latency degradation on a specific host, a Thread Profiler JSON export is gold for an AI agent—LLMs excel at parsing nested JSON call trees to pinpoint the exact hotspot function and library causing the bottleneck.
- **Eliminate External Causes First:** Before recommending or triggering a thread profile, your agent should rule out external bottlenecks:

1. **Database spans normal + External API spans normal + Host CPU high** → Strong signal of an **in-code bottleneck** (infinite loop, regex backtracking, heavy serialization, or GC thrashing). This is when code profiling is justified.
2. **Host CPU low + High endpoint latency** → Likely **thread blocking / lock contention / I/O wait** rather than pure CPU burn (remember that blocked threads still show up in the call tree).

- **Cross-Check Language Support:** If an alert fires on a **Go, Node.js, or PHP** service, your agent should know the classic APM Thread Profiler is unavailable for those runtimes and pivot instead to Distributed Traces, transaction breakdowns, runtime metrics (e.g., Node.js event loop lag / Go goroutine counts), or Continuous Profiling where supported.
