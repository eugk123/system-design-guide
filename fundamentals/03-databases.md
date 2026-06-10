# Databases: Storage, Replication, Sharding, Transactions

The storage layer is where most designs live or die. Picking the right store, the right
schema, and the right partition key is usually the highest-leverage decision you make.

## SQL vs NoSQL

### Relational (SQL) — Postgres, MySQL
- Structured schema, rows & columns, **joins**, **ACID transactions**, rich queries (SQL).
- Strong consistency by default; great when relationships and integrity matter.
- Scales **vertically** easily; scales **horizontally** with effort (read replicas, then
  sharding). Joins across shards are painful.
- Default choice unless you have a specific reason to leave. "I'd start with Postgres" is
  rarely wrong.

### NoSQL families
- **Key-value** (Redis, DynamoDB, Memcached): O(1) get/put by key. Caching, sessions, simple lookups.
- **Wide-column** (Cassandra, HBase, Bigtable): rows with flexible columns, partitioned by
  key, optimized for huge write throughput and horizontal scale. Tunable consistency.
- **Document** (MongoDB, Couchbase): JSON-like documents, flexible schema, good for
  hierarchical/aggregate data accessed as a unit.
- **Graph** (Neo4j): nodes + edges, fast relationship traversal (social graph, fraud rings).
- **Time-series** (InfluxDB, Prometheus), **Search** (Elasticsearch): specialized.

### When to pick NoSQL
- Massive scale / write throughput beyond a single primary.
- Flexible or evolving schema.
- Simple access patterns (lookup by key) that don't need joins.
- Willing to trade strong consistency for availability/partition tolerance.

> Senior framing: it's not SQL **or** NoSQL — it's **polyglot persistence**. Use Postgres for
> the transactional core, Redis for caching, S3 for blobs, Elasticsearch for search, all in
> one system. Pick per access pattern.

## Indexing
- An index is a separate sorted structure (usually a **B-tree**) mapping column values → row
  locations. Turns O(n) scans into O(log n) lookups.
- **Cost**: every write must update every index → slower writes, more storage. Index for your
  read patterns; don't over-index.
- **Composite indexes** support multi-column filters; order matters (leftmost-prefix rule).
- **Covering index**: includes all columns a query needs → serve from the index alone, no row fetch.
- **LSM-tree** stores (Cassandra, RocksDB): writes go to an in-memory memtable + append-only
  log, flushed to sorted files (SSTables), compacted in background. Fast writes, reads may
  touch multiple SSTables (mitigated by Bloom filters). Contrast with B-trees (update in place,
  read-optimized). Know this tradeoff: **B-tree = read-optimized, LSM = write-optimized.**

## Replication

Keep copies of data on multiple nodes for **availability**, **read scaling**, and **durability**.

### Leader–follower (primary–replica)
- Writes go to the **leader**; reads can go to **followers**.
- **Async replication**: leader acks before followers catch up → low write latency, but risk
  of data loss on leader failure and **replication lag** (stale reads on followers).
- **Sync replication**: wait for follower ack → no loss, higher latency. Often **semi-sync**
  (wait for one follower) as a compromise.
- **Read-your-own-writes** problem: a user writes, then reads from a lagging replica and
  doesn't see their change. Fix: route the user's reads to the leader for a window, or track a
  version/timestamp and read from a caught-up replica.

### Multi-leader / leader-less
- **Multi-leader**: accept writes in multiple regions → low write latency everywhere, but
  **write conflicts** need resolution (last-write-wins, CRDTs, app logic).
- **Leaderless** (Dynamo-style, Cassandra): any replica accepts reads/writes; use **quorums**.
  With N replicas, require W writes + R reads to ack. **W + R > N guarantees overlap →
  strong-ish consistency.** Tune W/R for the read/write balance you want. Repair via
  read-repair and anti-entropy (Merkle trees).

## Sharding (horizontal partitioning)

Split data across nodes when it no longer fits / can't be served by one machine.

### Strategies
- **Range partitioning**: by key range (A–F, G–M…). Supports range scans, but risks
  **hotspots** (e.g. time-ordered keys all hit the newest shard).
- **Hash partitioning**: `hash(key) % N` or consistent hashing. Even distribution, but loses
  range-scan ability.
- **Directory/lookup-based**: a lookup service maps key → shard. Flexible (can rebalance), but
  the lookup is another component and potential SPOF.
- **Geographic**: partition by region (also helps latency & data residency).

### Choosing the shard key — the crux decision
A good shard key:
1. **Distributes evenly** (no hotspots). Avoid low-cardinality or monotonically increasing keys.
2. **Co-locates data you query together** (so common queries hit one shard).
3. **Matches the dominant access pattern.** Shard a chat app by `conversation_id` so a
   conversation's messages live together; shard a feed by `user_id`.

### Costs of sharding
- **Cross-shard queries / joins** become scatter-gather (slow, complex). Denormalize to avoid.
- **Cross-shard transactions** need 2-phase commit or saga patterns — avoid if you can.
- **Rebalancing/resharding** is operationally hard. Use **consistent hashing** and/or many
  small **virtual shards** mapped to physical nodes so you can move shards without rehashing
  everything. Pre-split generously.
- **Hot shard / celebrity problem**: one key gets disproportionate traffic. Mitigate by
  further splitting that key, caching it, or special-casing it.

## Transactions & ACID
- **Atomicity** (all-or-nothing), **Consistency** (valid state transitions), **Isolation**
  (concurrent txns don't interfere), **Durability** (committed = survives crashes).
- **Isolation levels** (weakest→strongest): Read Uncommitted → Read Committed → Repeatable
  Read → Serializable. Higher = fewer anomalies (dirty/non-repeatable reads, phantoms) but
  more locking/contention.
- **Optimistic concurrency** (version/compare-and-set, no locks; retry on conflict) vs
  **pessimistic** (locks held during txn). Optimistic suits low-contention; pessimistic suits
  high-contention hotspots.

### Distributed transactions
- **2-Phase Commit (2PC)**: coordinator asks all to prepare, then commit. Strong, but blocking
  and a coordinator SPOF; poor availability under partitions. Avoid at scale.
- **Saga pattern**: a sequence of local transactions with **compensating actions** to undo on
  failure. Eventual consistency, no global lock. Preferred for microservices (e.g. order →
  payment → inventory, each with a rollback step).
- **Idempotency** everywhere so retries are safe.

## BASE vs ACID
NoSQL systems often favor **BASE** (Basically Available, Soft state, Eventually consistent):
accept staleness for availability and scale. Match the model to the data — money is ACID, a
"view count" is BASE.

## Interview signals
- You default to a relational store and justify any move to NoSQL with access patterns + scale.
- You choose a shard key explicitly and defend it on distribution + query locality + hotspots.
- You name replication lag and read-your-writes, and give a concrete fix.
- You reach for sagas/idempotency instead of distributed 2PC at scale.
- You know B-tree vs LSM and why it matters for write-heavy systems.
