---
title: Observability
category: Observability
priority: must-know
status: learning
difficulty: medium
interview_ready: false
tags:
  - hld
  - observability
  - telemetry
  - monitoring
---

# Observability (Monitoring, Logging, Metrics, Tracing)

## 1. One-Line Definition
Observability is the ability to understand a system's internal state from its external outputs — structured logging for "what happened," metrics for "are we healthy," and tracing for "how did one request behave across services" — so you can answer why it behaves oddly without guessing.

## 2. Why Do We Need It?
Distributed systems fail in ways a unit test never saw: a slow dependency, a dropped event, a memory leak, a hot partition. You can't debug what you can't see. Monitoring tells you a symptom (CPU 100%); observability tells you *why* (requests are piling in queue X because dependency Y's timeout tuning broke). Without it, deployment is guesswork, incident response is archaeology, and every "weird" bug becomes a core dump. Observability is the difference between diagnosing and guessing.

## 3. Simple Intuition
A driver's dashboard (metrics: fuel, speed), the car's event log (logging: "ABS engaged at 03:12"), and a mechanic who can follow the drive-train path for *one specific trip* (tracing). You drive WITH the dashboard; when something's wrong you pull the log; but to answer "why did *that* order end up in the wrong warehouse," you follow that one order's route — the trace. Each tool answers a different question; together they make the car explainable.

## 4. What Happens Without It?
"Something is slow" with no data → guess-from-memory debugging, blame-shifting between teams, and re-deploys that don't fix anything. Outages go on for hours while people grep logs in three systems and stitch the story. The system is a black box: healthy-looking dashboards (all green) hiding a 90% error rate in an unmonitored path. Or an over-tuned alert storm where every page is useless (see golden-signals / alerting for the anti-pattern).

## 5. Core Idea
- **The three pillars:**
  - **Logs:** discrete events — "order created id=123 at 12:00". Should be *structured* (JSON, not "Order created successfully!!") and *centralized* (aggregated for search, not per-host files). Low-signal/high-volume; you sample or index selectively.
  - **Metrics:** numeric time-series — QPS, latency percentiles (p50/p99/p99.9), error rate, queue depth, CPU. Cheap, always-on, good for dashboards *and* alerts. Dimensions (endpoint, tenant, region) make them sliceable.
  - **Traces:** each request's path across services with spans (service, duration, child links). Answers cross-service questions: which hop is slow, where do errors happen. Requires instrumentation everywhere (see distributed-tracing).
- **The question each answers:** logs "what happened," metrics "is it happening now/trending," traces "which part of this request." Combined = the classic debugging flow: metric says slow → trace says which hop → log shows why.
- **Golden signals are the crown jewels:** latency, traffic, errors, saturation (see golden-signals). Keep a 4-pane dashboard per service.
- **Search vs analyze:** logs → search/ELK/Loki; metrics → Prometheus/Grafana/CloudWatch; traces → Jaeger/Zipkin/DD/OpenTelemetry. **OpenTelemetry** is the emerging standard-instrumentation layer — write once, export anywhere.
- **COST is part of design:** observability at scale is a data-engineering problem (sampling, retention tiers, cardinality control). High-cardinality metrics (per-user labels) wreck metric stores — use traces for that, not metrics.

## 6. Important Terminology

| Term | Simple Meaning |
|------|----------------|
| Log | One structured event record |
| Metric | Numeric time-series value (QPS, latency) |
| Trace / span | Request path across services / one hop |
| Cardinality | Number of unique label combos (keep low!) |
| Structured logging | Machine-parseable JSON records |
| Centralized logging | Aggregating all logs to one searchable store |
| Dashboard | Visual chart set for a service's health |
| Instrumentation | Code/agent emitting telemetry |
| OpenTelemetry | Vendor-neutral telemetry API + SDK |
| Tail latency | Slow tail of the latency distribution (p99.9+) |

## 7. Basic Architecture

```mermaid
flowchart LR
    App[Services] -->|logs| L[Log collector]
    App -->|metrics| M[Prometheus]
    App -->|traces (OTel)| T[Tracing backend]
    L --> ELK[Search: Elastic/Loki]
    M --> G[Grafana dashboards + alerts]
    T --> J[Jaeger/Cloud-specific]
    tag[Correlation: common request/span ID links all three]
```

## 8. Request or Data Flow
1. Request enters gateway → trace context injected (trace/span ID) → passed via headers to every hop.
2. Each hop logs structured events with the trace ID; histogram metrics increment request counters + latency buckets.
3. Aggregators ship telemetry to backends; dashboards show QPS/latency/errors; alert rules fire on thresholds/burn rate.
4. Debug flow: dashboard spike → drill into distribution → trace the slowest request → find its spans → read the failing span's log → root cause.

## 9. Practical Example
**Search outage (assumptions):** "search is slow" ticket at 11:00.
- Metrics show p99 latency 2.1s (was 120ms) since a 10:52 deploy → trace of slowest queries shows 3 of 6 spans are in `query-builder` (new filter code) → log of that span shows a Cartesian-join warning → root cause: new join fan-out → roll back, confirm p99 normal.
- Without tracing: six teams each blame their own queue; hours. With it: 20 minutes. That's observability ROI.

## 10. Scaling
- **Volume:** logs grow with traffic — sample debug logs, keep structured ones; tier retention (hot 7d / warm 90d / cold archive). Metrics: aggregate (avg/p95) to bound storage; traces: *sample* (head/edge sampling or tail latency sampling) — you can't trace everything at a million QPS.
- **Cardinality control:** labels must be bounded sets (endpoint, service, version); never free-form (user IDs!) in metric labels.
- **Shared pipeline:** centralize collectors as a service (agent → collector → backend) with backpressure; this pipeline is itself infrastructure to HA and size.

## 11. Reliability and Failure Scenarios

| Failure | Happens | Detection | Recovery | Trade-off |
|---------|---------|-----------|----------|-----------|
| Collector down | Telemetry gap | Heartbeat | Reconnect + buffer | disk buffering |
| Log volume spike | Indexing lag/no search | Index thrash | Drop/sample low-value | risk blind spots |
| Metric card. explosion | Store memory blow-up | Cardinality alerts | Reduce labels | visibility loss |
| Tracing sampling wrong | Missing slow tail | "% traced" metric | Tail sampling | cost |
| UTC/drift timestamps | Wrong ordering | — | NTP + ms precision | — |

## 12. Consistency and Correctness
Telemetry is eventually-consistent and lossy by design — don't build safety signals that require 100% delivery. Alerting decisions belong on **metrics** (reliable), not logs alone. Correlation (trace/request id) is the glue: without it, logs of one request are scattered across hosts and can't be joined. Make every log line carry {trace_id, span_id, service, host, timestamp}.

## 13. Performance
- Instrumentation overhead: sub-% at design-in budget (batching, sampling); with OpenTelemetry, exporters batch and drop overruns.
- Backends: Prometheus is pull-scalable; logs need robust ingestion; traces the samest-heavy.
- Never log per-request at debug level in prod (cost × traffic); make levels config-aware.

## 14. Security
- Logs carry PII/secrets "for debugging" — redact tokens, cards, emails at source (this is why structured + centralized matters). Encrypt at rest, restrict access (full logs are bigger risk than prod data sometimes).
- Don't log secrets/keys even masked in a "safe" obfuscation — the cardinal rule: never log what you encrypted as plaintext adjacent to keys.

## 15. Trade-Offs

| Choice | Advantages | Disadvantages | When to Use |
|--------|------------|---------------|-------------|
| Metrics only | Cheap, always-on | No why/context | Dashboards, alerts |
| Logs only | Full narrative | Expensive, unstructured without care | Forensic single-machine |
| + Tracing | Story per request | Instrumentation cost | Distributed systems |
| Full OTel | Vendor-neutral, one pipeline | New stack learning | Any modern platform |
| Sample traces | Cost control | Missing rare slow tail | Huge QPS |

## 16. Common Mistakes
- Metrics without usable cardinality (dashboards break under label explosion).
- Alerts on *symptoms only* — no golden signals, so every alert is "p99 high" without "who/what."
- Logging unstructured prose (cannot search/join) or logging PII raw.
- Tracing only one team's service → "you don't see MY slow hop" (traces must cover the critical path globally).
- No correlation IDs: "we have logs, we have metrics, we can't link them."

## 17. HLD vs LLD Boundary
HLD: observability strategy per service (which signals), pipeline topology (collectors/backends), retention/sampling budget, correlation contract (IDs), cardinality rules, dashboards per golden signal, alert philosophy. LLD: OpenTelemetry setup, structured logger fields, metric histograms, span annotations, exporter configs, alert queries.

## 18. Interview Questions

### Beginner
- What question do logs, metrics, and traces each answer?
- Why is per-user label on a metric a bad idea?

### Intermediate
- Walk the debugging flow for "API is slow" using all three pillars, with concrete dashboards and queries.
- Design the telemetry pipeline for 100k RPS (collectors, sampling, retention) with cost control.

### Advanced
- Cardinality explosion hits your metric store mid-incident. Design the defense (bounded labels, aggregation, uniqueness budget) and the recovery (triage dashboards).
- Design observability for a system you're *also* designing — call out how fresh evidence feeds back into SLOs vs RPO/RTO.

## 19. Interview Memory Summary

> [!abstract]- Interview Memory Summary

### Remember

- Three pillars: logs = events (what happened), metrics = health (how bad/trending), traces = path (where) — each answers a different question.
- Correlation (trace/request ID on every log line) is what makes the three linkable; without it, evidence is scattered.
- Golden signals are the crown-jewel metric set: latency (split success vs failure), traffic, errors, saturation.
- Design the costs in: log volume, metric cardinality, trace sampling — observability at scale is a data-pipeline problem.
- OpenTelemetry is the emerging standard instrumentation layer — write once, export anywhere.
- Alert on metrics (reliable); diagnose via traces → logs.

### 30-Second Explanation

Emit structured logs with trace IDs, histograms/metrics per golden signal, sampled traces per request — central pipeline, bounded labels, and dashboards per service; alerting on metrics, diagnosis via traces→logs.

### Interview Traps

- "We have dashboards, so we're observable" — dashboards are output; observability is the ability to answer unknown questions.
- Per-user labels in metrics — cardinality explosion that breaks the store.
- Alerts on symptoms only, no golden signals.
- Unstructured logging, or raw PII in logs.
- Tracing only one service — a blind spot on the critical path.

### Key Trade-Off

Observability trades engineering effort and pipeline cost for the ability to answer unknown "why" questions — you pay in telemetry volume, cardinality control, sampling, and instrumentation, and you get diagnosis in minutes instead of multi-team archaeology.

## 20. Related Concepts

### Prerequisites

- [[reliability|Reliability]]
- [[latency-vs-throughput|Latency vs Throughput]]

### Commonly Used Together

- [[distributed-tracing|Distributed Tracing]]
- [[golden-signals|Golden Signals]]
- [[sli-slo-sla|SLI / SLO / SLA]]
- [[retry-and-timeout|Retry and Timeout]]

### Advanced Concepts

- [[consumer-lag|Consumer Lag]] — queue depth/backpressure as a saturation signal in stream pipelines

Related planned topics (not authored yet): logging, alerting, sampling strategies, dashboards.

## 21. References
OpenTelemetry docs, Google SRE (observability, monitoring), Microsoft Azure observability guide. Verify current tool behavior pre-interview.

## 22. Active Recall

Test yourself before revealing each answer.

> [!question]- What question do logs, metrics, and traces each answer?
> Logs → "what happened" (discrete events); metrics → "is it healthy now and trending" (numbers over time); traces → "which part of this one request was slow or failed" (causal path). The classic flow: a metric says slow → the trace says which hop → its log says why.

> [!question]- Why is a per-user label on a metric a bad idea?
> Cardinality: each label combination creates a new series, so millions of users mean millions of series — the store's memory blows up and dashboards/queries degrade. Keep labels to bounded sets (endpoint, service, version, tenant tier) and push per-user/per-request detail into traces and logs, not metric labels.

> [!question]- Design the telemetry pipeline for 100k RPS with cost control.
> Instrument once with OpenTelemetry; agents/exporters batch and ship to collectors that buffer, sample, and apply backpressure. Logs: structured, debug sampled, tiered retention (hot 7d / warm 90d / cold archive). Metrics: histograms with bounded labels + aggregation (avg/p95). Traces: head sampling 1–5% with tail sampling to keep slow/error traces. Backends: Prometheus/Grafana for metrics, Jaeger/Tempo for traces, Loki/ELK for logs, sized and made HA like any other service.

> [!question]- P99 latency spikes right after a deploy. Walk the debugging flow using all three pillars.
> Dashboard/metrics show the p99 spike timed to the deploy → drill into the histogram → tail-trace the slowest requests → spans concentrate in a new component → open that span's correlated log line (via trace_id) → identify the query/join change → roll back → confirm p99 on the dashboard. That metric → trace → log chain is what correlation IDs buy.

> [!question]- What does high-cardinality instrumentation give you, and what does it cost?
> It gives sliceability (per endpoint, tenant, version) to localize problems fast. It costs storage, query latency, and dashboard fragility — million-series blowups. The trade: bounded labels + aggregation + sampling recovers most of the visibility at a fraction of the cost; the rare slow tail is then handled by trace sampling, not more labels.

> [!question]- Trace everything vs sample — when is each right?
> Full tracing is only affordable at low QPS or for critical money flows. At scale, sample: head-based for predictable cost, tail-based (keep slow/error) to preserve diagnostic value where the problems live. Two wrong calls: 100% sampling at low volume then forgetting to reduce as traffic grows (cost cliff), and sampling that keeps only fast requests (destroys the purpose).

> [!question]- The log collector goes down mid-incident. What's lost and how do you protect the path?
> New logs are no longer indexed — history stops at the buffer fill point, then drops. Metrics keep working (pull side), so keep alerts on metrics so the gap doesn't blind the pager; collectors should disk-buffer and reconnect, dropping low-value debug first. Remember telemetry is eventually-consistent by design — never build safety alarms that depend on 100% delivery.

> [!question]- Alert storm: everything paging, nothing actionable. Diagnose and fix.
> Alerting on symptoms only (CPU/memory), on averages, or without golden signals and burn-rate windows → noise and fatigue. Fix: alert on golden-signal SLO burn rates (real user impact), unify error definitions across services, give each service a 4-pane golden dashboard, and cut thresholds to user journeys rather than infra trivia.

> [!question]- "We have dashboards, so we're observable." Respond.
> Dashboards are the output; observability is the capacity to answer NEW questions about a system you didn't anticipate. Prove it by: structured correlated telemetry (trace IDs in logs), the ability to answer "why was THIS request slow," bounded-cardinality design, and alerting on golden signals — not by a green dashboard hiding an unmonitored path.

> [!question]- Interview scenario: design observability while also designing the system. How does fresh evidence feed decisions?
> Emit golden signals from day one; define a correlation contract (trace/request IDs in every line); derive SLIs/SLOs from the signals; alert on burn rate; and loop evidence back into the design — metrics/traces validate capacity and latency assumptions, SLO burn ranks features vs reliability, and RPO/RTO drills produce DR evidence. Observability is designed in, not bolted on.

## 23. When Should I Use This?

### Use it when

- The system spans multiple services and failures cross boundaries.
- You need to diagnose "why" beyond dashboards — unknown questions.
- SLOs and alerting need real user-facing signals (golden signals, burn rates).
- Incidents today take hours across teams (debugging archaeology).
- You're already adopting OpenTelemetry — instrument once, export anywhere.

### Avoid it when

- A single service with a trivial failure surface — full three-pillar is overhead; metrics-only may do.
- No budget for pipeline operations — observability at scale is a data-engineering cost.
- Correlation IDs aren't agreed — the three pillars can't be joined and the benefit collapses.
- Cardinality can't be bounded — the metric store will break.
- No one owns alerts/on-call — instrumentation without triage is still guessing.

### What problem does it solve?

Distributed systems fail in ways no one predicted, and a black box makes every incident a guessing / archaeology / blame-shifting session. The bottleneck is invisible internal state across many hops. Structured, correlated telemetry (logs/metrics/traces) through a central pipeline answers any "why" — metric → trace → log — in minutes, and feeds that evidence back into SLOs and design.

### What problem does it NOT solve?

It doesn't guarantee quality (instrumenting without fixing changes nothing); it doesn't bound cost unless cardinality/sampling/retention are designed in; it won't cross service boundaries unless every hop propagates context; and it isn't correctness infrastructure — traces are observational, never authoritative.

## 24. Decision Connections

Decisions that go together with observability:

- [[distributed-tracing|Distributed Tracing]] — the causal glue for cross-service "why"; depends on the correlation contract you set.
- [[golden-signals|Golden Signals]] — the minimum meaningful metric set the dashboards and alerts should be built on.
- [[sli-slo-sla|SLI / SLO / SLA]] — the signals feed SLIs and burn-rate alerting; observability makes SLOs enforceable.
- [[retry-and-timeout|Retry and Timeout]] — retry counts/amplification are exactly the telemetry that exposes storms.
- [[rate-limiter|Rate Limiter]] — limiter metrics (429s, per-key over-limit) are core operational signals.
- [[consumer-lag|Consumer Lag]] — queue-depth saturation sensing in message pipelines, feeding the same dashboards.
- [[latency-vs-throughput|Latency vs Throughput]] — the measurement vocabulary behind the latency/traffic signals.
- [[reliability|Reliability]] — observability is how reliability targets are measured and defended.

Decision tree:

```
System is a black box; symptoms come from users
    |
    +-- Need "what happened"?
    |      → structured logs (centralized, trace_id on every line)
    |
    +-- Need "are we healthy / trending"?
    |      → metrics on [[golden-signals|Golden Signals]] → dashboards + SLO burn-rate alerts
    |
    +-- Need "which hop was slow for THIS request"?
    |      → [[distributed-tracing|Distributed Tracing]] via OpenTelemetry
    |
    +-- Cost under control?
    |      → bounded labels, sampling (head/tail), tiered retention, HA collectors
    |
    +-- Decisions loop?
           → [[sli-slo-sla|SLI / SLO / SLA]] + evidence back into capacity and reliability choices
```