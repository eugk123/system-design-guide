# Tradeoffs Cheat Sheet

The recurring "X vs Y" decisions, condensed. For each: when to pick which, and the one-line
reason. Saying the tradeoff out loud — then *deciding* — is the senior signal.

## SQL vs NoSQL
- **SQL**: relationships, joins, transactions, strong consistency, moderate scale. *Default.*
- **NoSQL**: massive scale, high write throughput, flexible schema, simple key access, OK with
  eventual consistency. *Justify the move with access patterns + scale.*
- Reality: **polyglot** — SQL core + Redis cache + S3 blobs + ES search.

## Strong vs eventual consistency
- **Strong**: money, inventory, locks, unique constraints. Costs latency & availability.
- **Eventual**: feeds, counts, likes, caches, DNS. Cheap, available, scalable.
- **Decide per operation, not per system.**

## Vertical vs horizontal scaling
- **Vertical**: simpler, no code change, hard ceiling, SPOF. *Start here.*
- **Horizontal**: unlimited, fault-tolerant, needs statelessness + partitioning. *Scale here.*

## Fan-out on write vs read (feeds, notifications, group chat)
- **Write (push)**: precompute → fast reads; expensive writes; bad for celebrities/dormant users.
- **Read (pull)**: cheap writes; expensive reads; good for high-fan-out producers.
- **Hybrid**: push for normal, pull for celebrities. *The senior answer.*

## Cache write strategy
- **Write-through**: fresh cache, slower writes. Read-after-write workloads.
- **Write-back**: fast writes, risk of loss. Tolerant/high-volume data.
- **Write-around** + **cache-aside read**: avoid polluting cache with write-once data. *Common default.*

## Cache eviction
- **LRU**: temporal locality. *Default.* **LFU**: stable popularity. **TTL**: bound staleness.

## Message queue vs event log
- **Queue (SQS/RabbitMQ)**: consume-once work distribution; no replay.
- **Log (Kafka)**: retained, replayable, ordered per partition, multi-consumer fan-out. *Default
  for event-driven / streaming.*

## Delivery semantics
- **At-least-once + idempotent consumers**. *The practical default.* True exactly-once is mostly
  at-least-once + dedup.

## Sync vs async
- **Sync**: needed for the user's immediate response; simpler reasoning.
- **Async (queue)**: everything else — absorbs spikes, decouples, resilient. *Move slow work off
  the request path.*

## L4 vs L7 load balancing
- **L4**: fast, protocol-agnostic, no content routing.
- **L7**: HTTP-aware, path/header routing, TLS termination, stickiness. *Usually what you want.*

## REST vs gRPC vs GraphQL
- **REST**: public, cache-friendly, simple. **gRPC**: internal, high-throughput, typed, streaming.
- **GraphQL**: flexible aggregated reads for diverse clients; complex caching + query-cost risk.

## Replication: leader-follower vs leaderless
- **Leader-follower**: simple, strong-ish; read replicas scale reads; watch replication lag.
- **Leaderless (quorum, Dynamo)**: high availability, tunable (W+R>N); conflict resolution needed.

## Sharding key
Pick for: **even distribution** (no hotspots) + **query locality** (co-locate what's read
together) + **matches dominant access pattern**. Use **consistent hashing** + virtual nodes to
reshard cheaply. Watch the **hot/celebrity key**.

## Partitioning: range vs hash
- **Range**: supports scans; risks hotspots (sequential keys).
- **Hash**: even spread; loses range scans.

## B-tree vs LSM-tree
- **B-tree**: read-optimized, update-in-place. *Default RDBMS.*
- **LSM-tree**: write-optimized, append + compact. *Write-heavy (Cassandra/RocksDB).*

## Idempotency / retries
- Retry only **idempotent** ops; **exponential backoff + jitter**; cap attempts; **circuit
  breaker** to stop cascading failure; **timeouts on every remote call**.

## Fail open vs fail closed (rate limiter, auth)
- **Open**: availability over enforcement (user-facing limiter). **Closed**: protect a fragile
  backend / security-critical path. *State the choice.*

## 301 vs 302 (URL shortener)
- **301 permanent**: browser-cached, less load, lose analytics/control.
- **302 temporary**: every hit reaches you → analytics & control, more load.

## Consistency knobs (quorum)
- **W+R > N** → overlap → strong-ish reads. Lower W → fast writes; lower R → fast reads.

## Geospatial index
- **Geohash**: simple, store-agnostic, shardable (+ query neighbor cells). *Default.*
- **Quadtree**: adapts to density. **S2/Hilbert**: production-grade sphere geometry.

## Counting at scale
- Exact counters contend → **async aggregation** for tolerant counts (views/likes); **HyperLogLog**
  for approximate distinct counts; **Count-Min Sketch** for frequency.

## The meta-tradeoff
**Latency ↔ Consistency ↔ Availability ↔ Cost ↔ Complexity.** You can't max all. Name which you're
optimizing and which you're sacrificing — and *why* — for each major decision. That sentence is
what gets you the senior rating.
