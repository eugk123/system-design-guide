# Problem 9: Distributed Cache (design Redis / Memcached at scale)

Tests data structures + distributed-systems coordination directly. The crux: **consistent
hashing for placement, eviction, and the consistency/availability tradeoffs of a replicated
cache.** This one is "build the building block" rather than "use the building block."

## Step 1 — Functional requirements
1. `get(key)`, `put(key, value, ttl?)`, `delete(key)`.
2. Bounded memory with **eviction** when full.
3. **TTL** expiration.
4. Scale horizontally across many nodes; tolerate node failure.
**Non-goals (defer)**: rich data types, transactions, persistence-as-primary-store.

## Step 2 — Non-functional requirements
- **Sub-millisecond reads/writes** (in-memory; that's the point).
- **High availability** and **horizontal scalability** (add nodes without a full reshuffle).
- **Eventual consistency** is acceptable (it's a cache, not source of truth).
- Even load distribution; graceful behavior on node add/remove.

## Step 3 — Capacity estimation
Working set drives sizing: hot-item count × item size. e.g. 100M hot items × ~1 KB ≈ **100 GB**
→ doesn't fit one node → shard across ~5–10 nodes (with headroom + replication). QPS in the
hundreds of thousands to millions/s → in-memory + sharding required.

## Step 4 — API
```
get(key) -> value | null
put(key, value, ttl?) -> ok
delete(key) -> ok
```
Client library (smart client) knows the node topology and routes directly to the owning node →
no proxy hop.

## Step 5 — High-level design
```
App ⇄ Cache client (consistent-hash router) ⇄ Cache node cluster (sharded)
                                                  each node: hash map + eviction + TTL
                  Topology/membership: gossip or a coordination service (ZK/etcd)
```
- **Single node** = a big in-memory **hash map** (O(1) get/put) + an **eviction policy
  structure** + TTL handling.
- **Cluster** = many such nodes; keys partitioned across them.

## Step 6 — Deep dives

### Placement: consistent hashing (the core)
- Naive `hash(key) % N` remaps ~all keys when N changes → cache-wide miss storm. Use a
  **consistent hash ring**: key maps to the next node clockwise; adding/removing a node only
  moves ~1/N keys.
- **Virtual nodes**: each physical node occupies many ring positions → even distribution and
  support for heterogeneous capacity. This is the textbook reason to know consistent hashing.
- The **smart client** (or a proxy like twemproxy) computes the ring and routes directly.

### Eviction (single-node data structure)
- **LRU** via a **hash map + doubly linked list**: map key→node; on access move node to head;
  evict from tail when full. O(1) get/put/evict. This is the classic implementation to be able
  to sketch.
- Alternatives: **LFU** (frequency-based), **TTL**-driven. Mention LRU as default.

### TTL expiration
- **Lazy**: check expiry on access, drop if expired.
- **Active**: a background sampler periodically evicts expired keys (Redis samples random keys).
- Use both: lazy for correctness on read, active to reclaim memory from untouched expired keys.

### Replication & availability
- **Replicate each shard** (primary + replica(s)) so a node failure doesn't lose the whole
  shard's hot data and cause a DB stampede.
- **Async replication** (fast, may lose last writes on failover) is usually fine for a cache.
- On primary failure → promote a replica; the client updates topology (via gossip / coordination
  service). Brief unavailability or stale reads acceptable.

### Consistency
- It's a cache → **eventual consistency** is the norm. Replication lag means a replica read may
  be stale. Decide if reads go to primary only (consistent within a shard) or replicas (more
  read throughput, possible staleness).
- Invalidation on write (delete the key) keeps it roughly in sync with the source of truth.

### Membership & topology changes
- Nodes join/leave; the cluster must converge on the ring. Use **gossip protocol** (Redis
  Cluster) or a **coordination service (ZooKeeper/etcd)** as the source of truth for membership.
  Clients refresh topology on change.

### Hot key problem
- One key gets disproportionate traffic → overloads its shard. Mitigate: replicate the hot key
  to multiple nodes, add a small **client-side local cache** (near cache) in front, or split the
  value.

## Step 7 — Bottlenecks & failure modes
- **Node loss** → consistent hashing limits the blast radius to ~1/N keys; replication preserves
  hot data; DB sees only a small miss bump (not a stampede).
- **Resharding / scaling** → consistent hashing + virtual nodes make adding capacity cheap.
- **Hot key / hot shard** → replicate hot key, near-cache, split.
- **Thundering herd on miss** → request coalescing / single-flight at the client.
- **Memory pressure** → eviction + TTL; monitor hit ratio (the key health metric) and evictions.

## Key takeaways
- **Consistent hashing + virtual nodes** for placement and cheap rescaling — the headline.
- Single node = **hash map + LRU (map + doubly linked list)** + TTL (lazy + active).
- **Replicate shards**, accept eventual consistency, manage membership via gossip/coordination.
- This problem is mostly about *being* the building block the other problems *use* — know it cold.
