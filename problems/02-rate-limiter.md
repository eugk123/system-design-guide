# Problem 2: Distributed Rate Limiter

Tests algorithms + distributed-state coordination in a tight scope. The crux: the **counter
algorithm** and keeping it **consistent and fast across many servers**.

## Step 1 — Functional requirements
- Limit clients to **N requests per time window** (e.g. 100 req/min per user/API-key/IP).
- Reject (or queue) requests over the limit; return `429 Too Many Requests` with a
  `Retry-After` header.
- Support **different rules** per endpoint/tier (free vs paid).
**Non-goals**: billing, full WAF.

## Step 2 — Non-functional requirements
- **Very low latency** — it sits in front of *every* request; must add < ~1–2 ms.
- **High availability** — if the limiter is down, **fail open** (allow) vs **fail closed**
  (block). Usually **fail open** for user-facing (don't take an outage to enforce limits);
  fail closed for protecting a fragile/critical backend. State the choice.
- **Accuracy vs performance**: a little over-counting at boundaries may be acceptable for speed.
- **Distributed**: many gateway nodes must share a consistent view of each client's count.

## Step 3 — Where it lives
Typically at the **API gateway / edge / middleware** before requests reach app servers, so bad
traffic is rejected as early and cheaply as possible. The counters live in a **fast shared
store (Redis)** so all nodes agree.

## Step 4 — Algorithms (the core of the problem)

### 1. Fixed window counter
Count requests per fixed window (e.g. per minute, keyed `user:minute`). Increment; reject if >
limit; window resets each period.
- **Pro**: trivial, memory-cheap (one counter per key).
- **Con**: **boundary burst** — a client can send N at 0:59 and N at 1:00 → 2N in 2 seconds.

### 2. Sliding window log
Store a timestamp per request (sorted set). On each request, drop timestamps older than the
window, count the rest, allow if < limit.
- **Pro**: exact, no boundary problem.
- **Con**: memory = O(requests) per client; expensive at high volume.

### 3. Sliding window counter (recommended general default)
Approximate the sliding window using the current + previous fixed-window counts, weighted by
how far into the current window we are:
`count ≈ prevWindowCount × (overlap fraction) + currWindowCount`.
- **Pro**: smooths the boundary burst, O(1) memory (two counters), good accuracy. Great
  balance — what most production limiters use.

### 4. Token bucket (recommended when bursts are OK)
A bucket holds up to `B` tokens, refilled at `r` tokens/sec. Each request consumes a token;
empty bucket → reject.
- **Pro**: allows **controlled bursts** up to bucket size while enforcing a long-run average
  rate. Memory-cheap (store tokens + last-refill timestamp; refill lazily on access).
- Used by AWS, Stripe, etc. The most common production choice.

### 5. Leaky bucket
Requests enter a queue (bucket) drained at a fixed rate; overflow is dropped.
- **Pro**: smooths output to a **constant rate** (good for protecting a downstream that needs
  steady load). **Con**: no bursts; adds queueing latency.

> Default recommendation: **token bucket** (burst-friendly, cheap) or **sliding window counter**
> (smooth, accurate). Name the tradeoff: bursts allowed vs strictly smoothed.

## Step 5 — Distributed design
```
Request → Gateway node (rule lookup) → Redis (atomic counter check) → allow / 429
```
- Counters in **Redis** (in-memory, fast, shared). Key = `{clientId}:{rule}`; set a TTL = window.
- **Atomicity is essential**: read-modify-write across concurrent requests must be atomic or
  you'll over-allow under races. Use Redis **`INCR` + `EXPIRE`** atomically, or a **Lua script**
  (token bucket / sliding window logic runs server-side in one atomic step). Lua is the clean way.
- Rules/config in a config store, cached locally on each gateway and refreshed.

## Step 6 — Deep dives

### Reducing Redis latency/load
- One Redis round trip per request adds latency and centralizes load. Optimize with a
  **two-tier** approach: keep an approximate **local in-memory counter** per node and
  periodically sync/borrow quota from Redis. Trades a little accuracy for big latency/throughput
  wins.
- **Sharding Redis** by client key spreads counter load; hot clients can be a hot key (replicate
  or local-cap them).

### Distributed accuracy
- With per-node local counters, the global limit can be exceeded by up to (nodes × slack).
  Acceptable for protection; not for hard quotas. For strict limits, centralize in Redis and
  accept the latency, or allocate per-node sub-quotas (limit/N each).

### Race conditions
- Naive `GET` then `SET` races under concurrency → use atomic `INCR`/Lua. This is the classic
  bug to call out.

### Response & UX
- Return `429` + `Retry-After` + headers (`X-RateLimit-Limit/Remaining/Reset`) so clients can
  back off gracefully. Optionally **queue** instead of reject for some workloads (leaky bucket).

## Step 7 — Failure modes
- **Redis down** → fail open (allow) for availability, or degrade to local-only limiting.
  Decide and state it.
- **Hot client / DDoS** → also need coarse IP-level limiting upstream and edge DDoS protection;
  app-level rate limiting isn't a DDoS defense by itself.
- **Clock skew** across nodes → use Redis server time / TTLs rather than local wall clocks for
  windows.
- **Config changes** → propagate new rules without restart (config push + local cache TTL).

## Key takeaways
- Know the **five algorithms** and their tradeoffs cold; default to **token bucket** or
  **sliding window counter** and justify.
- The distributed crux is **atomic shared counters in Redis** (INCR/Lua) and the
  **accuracy-vs-latency** tradeoff of local vs centralized counting.
- Decide **fail-open vs fail-closed** explicitly.
