# Observability & Operations

Often skipped by candidates, which is exactly why mentioning it signals seniority. A design
isn't done when it works in the happy path — it's done when you can *run* it, *debug* it at
3am, and *prove* it meets its SLOs.

## The three pillars of observability
- **Metrics** — numeric time series (counters, gauges, histograms). Cheap, aggregatable,
  alertable. "How many / how fast / how full." Tools: Prometheus, StatsD, time-series DBs.
- **Logs** — discrete, structured events with context. "What exactly happened for this
  request." Structured (JSON) logs + centralized aggregation (ELK, Loki). Sample high-volume
  logs to control cost.
- **Traces** — follow one request across services end-to-end (spans with timing). Find *where*
  latency or errors come from in a distributed call graph. Tools: OpenTelemetry, Jaeger, Zipkin.
  Propagate a **trace/correlation ID** through every hop.

Metrics tell you *something is wrong*; traces tell you *where*; logs tell you *why*.

## The golden signals (what to alert on)
- **Latency** — p50 / p95 / p99 / p99.9. **Always look at tail latency, not averages** — the
  mean hides the users having a terrible time. Tail latency is amplified by fan-out (a request
  hitting 100 services waits on the slowest of 100).
- **Traffic** — requests/sec, throughput.
- **Errors** — error rate (5xx, failed jobs), by type.
- **Saturation** — how full the resources are (CPU, memory, disk, queue depth, connection pools).

(For queues/streams specifically: **consumer lag** and **queue depth** are the key health signals.)

## SLI / SLO / SLA
- **SLI** (Indicator): a measured quantity, e.g. "% of requests < 200ms" or "success rate."
- **SLO** (Objective): the internal target for an SLI, e.g. "99.9% of reads < 200ms over 30d."
- **SLA** (Agreement): the external, contractual promise (with penalties). SLA ≤ SLO always —
  keep internal targets stricter than what you promise customers.
- **Error budget**: 100% − SLO. If your SLO is 99.9%, you have 0.1% to "spend" on
  failures/risky deploys. Frames the tension between reliability and velocity: budget left →
  ship faster; budget burned → freeze and stabilize.

## Health checks & self-healing
- **Liveness** (is the process alive — restart if not) vs **readiness** (is it ready to serve
  traffic — pull from LB if not, e.g. during warmup). Distinguish them.
- Orchestrators (Kubernetes) restart unhealthy pods, reschedule on node failure, and gate
  rollouts on readiness.

## Alerting that doesn't burn people out
- Alert on **symptoms users feel** (SLO burn, error rate, latency), not every internal blip.
- **Page** for urgent/actionable; **ticket/dashboard** for the rest. Every page should be
  actionable and have a runbook.
- Avoid alert fatigue — noisy alerts get ignored, and then the real one gets missed.

## Capacity & cost
- Track utilization vs provisioned capacity; plan headroom (run <70%, see scaling doc).
- **Cost is a design constraint** at scale: storage tiering (hot/warm/cold → S3/Glacier),
  compression, retention policies, right-sizing instances, spot/preemptible for batch work.
  A senior answer weighs $ alongside latency and availability.

## Operational practices
- **Runbooks** for common failures; **on-call** rotations with clear escalation.
- **Postmortems** — blameless, focused on systemic fixes, with action items.
- **Chaos engineering** — deliberately inject failures (kill nodes, add latency, partition the
  network) to verify resilience before reality tests it for you.
- **Load testing** — validate capacity assumptions against the numbers you estimated.

## Security & privacy (adjacent, worth a mention)
- **Encryption** in transit (TLS) and at rest. **Secrets management** (vault, not in code).
- **AuthN/AuthZ** at the gateway; principle of least privilege between services.
- **PII handling**: minimize, encrypt, control access, support deletion (GDPR/data residency).
  Relevant when the design touches user data — and a strength area to lean into.
- **Rate limiting / WAF / DDoS protection** at the edge.

## Interview signals
- You mention metrics/logs/traces and propagate a correlation ID across services.
- You reason in p99/tail latency, not averages, and note fan-out amplifies the tail.
- You frame reliability with SLOs + error budgets rather than vague "high availability."
- You bring up cost/storage tiering and data privacy when the design warrants it.
- You distinguish liveness vs readiness and describe how the system self-heals.
