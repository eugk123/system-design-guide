# The One-Page Crammer

Everything that matters, on one page. Read this the night before. Everything here is expanded
elsewhere in the guide.

## The 7-step framework (drive every interview with this)
1. **Functional reqs** — core use cases + non-goals. Read vs write heavy?
2. **Non-functional reqs** — scale, latency (p99), availability, **consistency posture (CAP)**, durability.
3. **Estimate** — QPS (avg + peak), storage, bandwidth. **Compute read:write ratio.**
4. **API** — a few endpoints; idempotency keys, cursor pagination.
5. **High-level design** — boxes; walk one write path + one read path. Start simple.
6. **Deep dive** — data model, **shard key**, the bottleneck, the problem's unique twist.
7. **Bottlenecks & failure modes** — SPOFs, hot keys, 10× scale, retries/circuit breakers.

## Senior signals (the rubric)
- **Drive** the problem. **Quantify** every decision. **Name the tradeoff, then pick.**
- Design the **unhappy path** (failures, partitions, spikes). **Evolve** the design under scale.
- Pair each decision with its **cost**: "I'll use X; cost is Y; I accept it because Z."
- Boring tech by default (Postgres/Redis/S3/queue); justify exotic tech with numbers.

## Numbers (memorize)
- RAM ~100 ns; SSD read 1MB ~1 ms; **disk seek ~10 ms**; **same-DC RTT ~0.5 ms; cross-region ~150 ms**.
- RAM ≈ **100,000×** faster than disk seek → cache. Cross-region calls kill latency budgets.
- **86,400 s/day ≈ 10^5.** **1M/day ≈ 12/s. 1B/day ≈ 12K/s.**
- Peak = avg × (2–10×, state it). Sizes: row ~1 KB, photo ~1 MB, video minute ~10s of MB.
- Nines: 99.9% = 8.8 h/yr, 99.99% = 52 m/yr. Series deps multiply.

## Building blocks → reach for
| Need | Use |
|------|-----|
| Spread traffic | LB (L7), DNS/anycast |
| Speed reads | Cache (Redis), CDN, read replicas |
| Decouple / absorb spikes | Queue (SQS) / log (Kafka) |
| Relational | Postgres → replicas → shard |
| Huge/simple/write-heavy | Cassandra/DynamoDB |
| Blobs | S3 + CDN |
| Search | Elasticsearch |
| Coordinate/lock/elect | ZooKeeper/etcd (Raft) |
| Unique IDs | Snowflake / KGS pool |
| Geo | Geohash / quadtree / S2 |
| Count at scale | Async aggregation; HLL; Count-Min Sketch |

## Core tradeoffs (one-liners)
- **SQL** (joins/txns, default) vs **NoSQL** (scale/flex/throughput).
- **Strong** (money/locks/inventory) vs **eventual** (feeds/counts/cache) — *decide per operation*.
- **Vertical** (start) vs **horizontal** (scale; needs statelessness + partitioning).
- **Fan-out write** (fast reads) vs **read** (cheap writes) → **hybrid for celebrities**.
- **Cache-aside + write-through/invalidate**; eviction **LRU**.
- **Queue** (consume-once) vs **log/Kafka** (retained, replayable, multi-consumer).
- **At-least-once + idempotent consumers** = effective exactly-once.
- **B-tree** (read) vs **LSM** (write). **Range** (scans) vs **hash** (even) partitioning.
- **Consistent hashing + virtual nodes** for caches/shards (cheap rescaling).
- Retries: **only idempotent**, **backoff + jitter**, **circuit breaker**, **timeouts everywhere**.

## Shard key checklist
Even distribution (no hotspots) • query locality (co-locate what's read together) • matches
dominant access pattern • plan resharding (consistent hashing) • beware the **hot/celebrity key**.

## Failure-mode reflexes
SPOF → replicate. Hot key → replicate/split/near-cache. Stampede → single-flight + jittered TTL.
Cascading failure → timeout + circuit breaker + capped jittered retry + bulkhead + load shed.
Overload → graceful degradation (serve stale, drop non-essential). At-least-once → dedup by ID.

## The 14 problems — one-line crux each
1. **URL shortener** — unique short ID (KGS pool / Snowflake, not a naive counter); cache reads.
2. **Rate limiter** — token bucket / sliding window; **atomic Redis counter (INCR/Lua)**; fail-open.
3. **News feed** — fan-out push vs pull → **hybrid**; **celebrity** = pull; store postIds; async fanout.
4. **Chat** — WebSockets + **stateful gateways + session registry**; persist-before-deliver; partition by conversationId (ordering). Presence is the trap.
5. **Notifications** — per-channel **queues**; at-least-once + idempotency + DLQ + provider failover; anti-spam/prefs.
6. **Typeahead** — **trie with precomputed top-K per node**; split **serving (in-mem) vs build (offline)**; debounce + edge cache.
7. **Video** — **blob store + CDN + async transcoding**; **adaptive bitrate (HLS/DASH)**; bandwidth = cost.
8. **Proximity** — normal index fails → **geohash (+ neighbor cells)** / quadtree / S2; Uber = write firehose in memory.
9. **Distributed cache** — *be Redis*: **consistent hashing + vnodes**, LRU (map+DLL), TTL, replicate shards.
10. **Web crawler** — distributed **BFS**; **frontier** (priority + per-host politeness); dedup (Bloom + content hash); partition by host.
11. **Dropbox** — **metadata (DB) vs content (blobs)**; **chunking + content-addressed dedup + delta sync**; cursors + conflict copies.
12. **Payment** — correctness > all; **idempotency keys end-to-end**; **double-entry ledger**; persist-before-PSP; treat "unknown" as pending; reconcile.
13. **Ad aggregator** — **Kafka + windowed stream processing**; **Lambda (stream=fresh, batch=exact)**; exactly-once counting; HLL/CMS.
14. **Ticketmaster** — **never oversell** (strong consistency); **atomic hold + TTL** (Redis NX/EX + fencing); **virtual waiting room** for flash sale.

## Recurring "twists" by category
- **Fan-out** (feed, notifications, group chat) → push/pull/hybrid + hot-key handling.
- **Limited inventory** (tickets, payments) → strong consistency, idempotency, atomic transitions.
- **Firehose writes** (ad clicks, Uber locations) → buffer (Kafka) + in-memory/stream + approximate where OK.
- **Huge blobs** (video, Dropbox) → object store + CDN + chunk/transcode async; metadata separate.
- **Real-time delivery** (chat) → persistent connections + stateful gateways + persist-first.
- **Specialized index** (typeahead=trie, proximity=geohash, search=inverted index).

## Final reminders
- Restate assumptions out loud (so errors are catchable). Manage the clock.
- If you spot your own mistake, fix it aloud — that's a *positive* signal.
- Close with a 30-sec recap: design + the 1–2 key tradeoffs + what you'd ship as v1.
