# System Design Guide

A self-study guide to brush up on system design fundamentals and to solve canonical
problems under interview / promo-panel conditions. Written with a **senior+ (IC6) lens**:
the emphasis is on *tradeoffs and judgment*, not memorizing a single "right" architecture.

## How to use this guide

1. **Start with the framework** (`00-interview-framework.md`). Internalize the 7-step loop.
   Everything else hangs off it.
2. **Refresh fundamentals** (`fundamentals/`). Read in order the first time; later use them
   as reference. Each ends with "Interview signals" — the points that distinguish a senior
   answer from a junior one.
3. **Work the problems** (`problems/`). For each one: cover the solution, give yourself
   45 minutes on a whiteboard / blank doc, *then* compare. Reading solutions is not practice.
4. **Drill the reference** (`reference/`). Memorize the latency numbers and the
   tradeoff cheat-sheet. These are the things you should never have to derive live.

## What separates a senior answer

The bar at IC6 is not "can you draw boxes and arrows." It's:

- **Drive the problem.** You scope it, you propose, you steer. You don't wait to be fed reqs.
- **Quantify.** Every design decision is backed by a number (QPS, storage, latency budget).
- **Name the tradeoff, then pick.** "I'll use X. The cost is Y. I accept that because Z."
  Indecision reads as inexperience.
- **Know the failure modes.** What happens when this node dies? When traffic 10×'s? When
  the network partitions? Seniors design for the unhappy path.
- **Evolve the design.** Start simple, then scale it under pressure ("now we have 100M users").
  Show you can grow a system, not just draw its final form.

## Contents

### Framework
- `00-interview-framework.md` — the repeatable 7-step approach + how to manage 45 minutes

### Fundamentals
- `fundamentals/01-back-of-envelope.md` — estimation, the numbers to know
- `fundamentals/02-networking-and-apis.md` — DNS, load balancing, API styles, protocols
- `fundamentals/03-databases.md` — SQL vs NoSQL, indexing, replication, sharding, transactions
- `fundamentals/04-caching.md` — cache strategies, eviction, invalidation, CDNs
- `fundamentals/05-messaging-and-streaming.md` — queues, Kafka, pub/sub, delivery semantics
- `fundamentals/06-consistency-and-consensus.md` — CAP, PACELC, consistency models, Raft
- `fundamentals/07-scaling-and-availability.md` — partitioning, replication, failover, SLAs
- `fundamentals/08-observability-and-ops.md` — metrics, logs, traces, deploys, capacity

### Worked problems
- `problems/01-url-shortener.md` — unique ID generation, read-heavy scaling
- `problems/02-rate-limiter.md` — counter algorithms, atomic distributed state
- `problems/03-news-feed.md` — fan-out push vs pull, celebrity problem
- `problems/04-chat-system.md` — WebSockets, stateful gateways, ordering
- `problems/05-notification-system.md` — queue fan-out, reliability, anti-spam
- `problems/06-typeahead-autocomplete.md` — trie, serving vs build split
- `problems/07-video-streaming.md` — blob store, CDN, transcoding pipeline
- `problems/08-proximity-service.md` — geospatial indexing (geohash/quadtree)
- `problems/09-distributed-cache.md` — *be* Redis: consistent hashing, eviction
- `problems/10-web-crawler.md` — distributed BFS, frontier, dedup
- `problems/11-file-storage-sync.md` — Dropbox: chunking/dedup, sync, metadata vs content
- `problems/12-payment-system.md` — correctness-first: idempotency, ledger, reconciliation
- `problems/13-ad-click-aggregator.md` — stream processing, Lambda arch, exactly-once counting
- `problems/14-ticketmaster.md` — inventory concurrency, holds+TTL, flash-sale waiting room

### Reference
- `reference/numbers-to-know.md` — latency table, capacity rules of thumb
- `reference/tradeoffs-cheatsheet.md` — the recurring "X vs Y" decisions, condensed
- `reference/one-page-crammer.md` — the 48-hours-before cram sheet
