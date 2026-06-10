# Problem 6: Typeahead / Search Autocomplete

Tests data structures (trie) + extreme read latency + a build pipeline. The crux: **serving
ranked prefix suggestions in single-digit milliseconds, and the offline pipeline that feeds it.**

## Step 1 — Functional requirements
1. As the user types a prefix, return the **top K** (e.g. 5–10) suggestions.
2. Suggestions ranked by **popularity** (frequency), possibly personalized/recent.
3. Reflect trending queries over time.
**Non-goals (defer)**: spell-correction, full search results, deep personalization.

## Step 2 — Non-functional requirements
- **Extremely low latency** — fires on **every keystroke**; must be < ~50–100 ms end-to-end,
  ideally < 10 ms server-side. This dominates the design.
- **Read-heavy** by a huge margin (keystrokes ≫ new queries).
- **High availability**; **eventual consistency is fine** (suggestions can lag reality by hours).
- Scale: billions of queries/day → enormous keystroke QPS.

## Step 3 — Capacity estimation
~5B searches/day, ~20 chars avg, a request every few keystrokes (debounced) → effectively
**hundreds of thousands of suggestion QPS, peak ~millions/s**. Storage for the dictionary of
distinct queries: millions–billions of phrases → but the *served* structure is the top
suggestions per prefix, which is far smaller and cacheable.

## Step 4 — API
```
GET /suggest?q={prefix}&limit=10  -> { suggestions: [ {text, score}, ... ] }
```
Client **debounces** (e.g. wait ~50–100ms after a keystroke) and cancels in-flight requests to
cut load. Mention this — it's a real, expected optimization.

## Step 5 — High-level design (two halves)
This problem splits cleanly into a **serving path** (fast, online) and a **build path**
(offline, slow). Drawing this separation is the key structural insight.

```
[ Serving ]  Client → CDN/edge cache → Suggest service → Trie store (in-memory) 
[ Build  ]   Query logs → aggregation (count, time-decay) → build/rank → publish trie snapshot
```

## Step 6 — Deep dives

### The data structure: trie (prefix tree)
- A **trie** maps prefixes → completions. Each node = a character; paths = prefixes.
- Naive trie requires traversing all descendants of a prefix to find the top K → too slow.
- **Optimization: precompute and cache the top-K suggestions *at each node*.** Then a lookup is
  "walk to the prefix node, return its stored top-K" → O(prefix length), basically O(1) for
  short prefixes. This is the core trick.
- Keep the trie **in memory** for speed; it's read-mostly and rebuilt offline.

### Serving path
- Suggest service holds the trie in RAM (or queries an in-memory store). Lookup = traverse to
  prefix node → return cached top-K.
- **Aggressive caching**: popular prefixes' results are cached at the **CDN/edge** and in
  app-level cache. Most keystrokes hit cache, never the trie service. Hit ratio is very high
  because prefixes are heavily skewed (everyone types "fa", "the", etc.).

### Build path (offline pipeline)
- **Collect**: log every executed search query to a stream (Kafka).
- **Aggregate**: batch/stream job counts frequencies over a window, applies **time decay**
  (recent queries weighted higher → trending) and filters (min frequency, profanity/safety).
- **Build**: construct the trie with top-K precomputed per node.
- **Publish**: ship the new trie as an immutable **snapshot** to serving nodes (atomic swap).
  Rebuild cadence: hourly/daily for the bulk; a faster path for trending terms if needed.
- This keeps the expensive ranking work **off the serving path** entirely.

### Scaling the trie (sharding)
A global trie can be huge. **Shard by prefix** (e.g. by first 1–2 characters) across nodes;
route a request to the shard owning its prefix. Replicate each shard for read scale + HA. Small
enough tries can just be fully replicated to every node (simplest if it fits in RAM).

### Ranking
- Base score = frequency with time decay. Can layer in personalization (user history, location)
  and demotion of stale/unsafe terms. Keep heavy ranking in the offline build; the serving path
  only reads precomputed scores.

### Freshness vs cost
- Real-time trending requires faster rebuilds / a separate hot-term layer; full freshness is
  expensive. State the tradeoff: most autocomplete is fine with hourly/daily rebuilds + edge
  caching; add a lightweight real-time layer only for trending if required.

## Step 7 — Bottlenecks & failure modes
- **Latency** → in-memory trie with precomputed top-K + multi-layer caching (edge + app);
  client debounce. This is the whole point.
- **Hot prefixes** → naturally cached at the edge (skew helps you here).
- **Trie too big for one node** → shard by prefix; replicate shards.
- **Build pipeline lag/failure** → serving keeps using the last good snapshot (immutable
  snapshots make this safe); rebuild is non-urgent.
- **Bad/unsafe suggestions** → filtering in the build stage; denylist; the offline pipeline is
  where safety is enforced.
- **Cold start after deploy** → load the snapshot before taking traffic (readiness check).

## Key takeaways
- **Split serving (fast, in-memory trie with precomputed top-K) from building (offline
  aggregation + ranking + snapshot publish).** That separation is the structural insight.
- **Precompute top-K at each trie node** so a lookup is O(prefix length); cache hard at the edge.
- Eventual consistency + immutable snapshots make the system both fast and operationally safe.
- Don't forget the **client-side debounce** — it's part of the design.
