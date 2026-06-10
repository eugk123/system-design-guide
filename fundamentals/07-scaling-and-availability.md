# Scaling, Availability & Resilience

How to grow a system and keep it up. The framework's final steps (bottlenecks, failure modes)
draw heavily on this.

## Vertical vs horizontal scaling
- **Vertical (scale up)**: bigger machine (more CPU/RAM). Simple, no code change, but a hard
  ceiling and a single point of failure. Good early; expensive and finite later.
- **Horizontal (scale out)**: more machines behind a load balancer. Effectively unlimited,
  fault-tolerant, but requires statelessness, partitioning, and coordination. The path to
  serious scale.
- **Rule**: scale **up** until it's uneconomical or risky, then scale **out**. Design app
  servers stateless from day one so scaling out is just "add nodes."

## Statelessness enables scale
Stateless services scale trivially (any node serves any request; failures lose nothing).
Externalize state to shared stores (DB, Redis, object storage) or signed tokens. Keep the
*stateful* parts (databases, stateful gateways) small and well-managed.

## Reading scale vs writing scale
- **Read scaling**: caches, read replicas, CDNs, denormalized read models. Usually the easy
  direction.
- **Write scaling**: sharding/partitioning, async processing/queues, write-optimized stores
  (LSM), batching, splitting write-heavy entities. The harder direction — most "hard scaling"
  problems are write problems.

## Redundancy & eliminating SPOFs
A **single point of failure** is any component whose loss takes down the system. Find them and
add redundancy:
- **Active-passive (failover)**: a standby takes over when the primary dies. Simpler; standby
  capacity idle. Watch the **failover time** and split-brain risk.
- **Active-active**: all replicas serve traffic; on failure, others absorb load. Better
  utilization and instant failover, but needs conflict handling / shared state.
- Apply to every tier: multiple LBs, multiple app servers, DB replicas, multi-AZ, multi-region.

## Multi-region / geo-distribution
- **Why**: lower latency (serve near users), disaster recovery, data residency/compliance.
- **Active-passive (DR)**: one region serves, another on standby. Simpler; RPO/RTO tradeoffs.
- **Active-active**: all regions serve; needs cross-region replication and conflict resolution
  (multi-leader or partition-by-region). Hardest but best latency + availability.
- **Data placement**: keep a user's data near them (geo-sharding). Cross-region calls (~100+ms)
  are expensive — minimize synchronous cross-region dependencies on the request path.
- Define **RPO** (how much data you can afford to lose) and **RTO** (how fast you must recover).

## Availability math (the nines)
| Availability | Downtime/year | Downtime/day |
|--------------|---------------|--------------|
| 99% (two 9s) | ~3.65 days | ~14.4 min |
| 99.9% (three 9s) | ~8.77 hours | ~1.44 min |
| 99.99% (four 9s) | ~52.6 min | ~8.6 sec |
| 99.999% (five 9s) | ~5.26 min | ~0.86 sec |

- **Series dependencies multiply**: if a request needs 3 services each at 99.9%, end-to-end ≈
  99.9%^3 ≈ **99.7%**. More dependencies = lower combined availability → minimize critical-path
  dependencies, add redundancy, and make non-critical deps async/optional.
- **Redundancy improves it**: two independent 99% components in parallel ≈ 1 − (0.01)² = 99.99%.

## Resilience patterns (designing for the unhappy path)
- **Retries with exponential backoff + jitter**: retry transient failures, but back off and
  randomize so you don't synchronize a retry storm. Cap attempts. **Only retry idempotent ops.**
- **Circuit breaker**: after N consecutive failures, "open" the circuit and fail fast (or serve
  fallback) instead of piling onto a sick dependency; periodically "half-open" to test recovery.
  Prevents cascading failure.
- **Timeouts**: every remote call needs one. No timeout = a slow dependency exhausts your
  threads and takes you down. Set them aggressively.
- **Bulkheads**: isolate resources (thread pools/connection pools) per dependency so one
  failing dependency can't consume all capacity.
- **Graceful degradation**: shed non-essential features under load (serve stale cache, hide
  recommendations, disable expensive widgets) instead of failing entirely.
- **Load shedding / rate limiting**: reject excess load early to protect the core. Prioritize
  important traffic.
- **Backpressure**: signal upstream to slow down rather than collapsing.
- **Idempotency**: the prerequisite that makes retries and at-least-once delivery safe.

## Cascading failures & how they happen
A common outage shape: a dependency slows → callers' threads block on it → callers exhaust
threads → callers appear down → *their* callers retry aggressively → the retry storm finishes
off the recovering dependency. Defenses: timeouts + circuit breakers + capped jittered retries
+ bulkheads + load shedding. Mention this chain — it shows real operational experience.

## Capacity planning & autoscaling
- Provision for **peak**, not average; keep **headroom** (e.g. run at <70% so a node loss
  doesn't tip the rest over).
- **Autoscaling**: scale on a leading signal (CPU, queue depth, RPS). Beware scale-up lag —
  pre-warm for predictable spikes (flash sales, launches).
- **N+1 / N+2 redundancy**: enough spare capacity to lose 1 (or 2) nodes and still serve peak.

## Deployment safety (ties to availability)
- **Blue-green**: stand up the new version alongside old, switch traffic, roll back instantly.
- **Canary**: send a small % of traffic to the new version, watch metrics, ramp up.
- **Rolling**: replace instances gradually.
- **Feature flags**: decouple deploy from release; kill-switch risky features without redeploy.
All exist to limit blast radius — a deploy is the most common cause of outages.

## Interview signals
- You design app tiers stateless and identify SPOFs at every layer, adding redundancy.
- You compute combined availability and minimize critical-path dependencies.
- You apply timeouts + circuit breakers + jittered backoff and explain cascading failure.
- You provision for peak with headroom and define RPO/RTO for multi-region.
- You reach for graceful degradation / load shedding instead of hard failure under overload.
