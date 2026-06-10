# Numbers to Know (Cheat Sheet)

Memorize these. You should never burn interview time deriving them.

## Latency ladder (orders of magnitude)
| Operation | Time | Relative |
|-----------|------|----------|
| L1 cache | ~1 ns | — |
| L2 cache | ~4 ns | — |
| Mutex lock/unlock | ~17 ns | — |
| Main memory (RAM) | ~100 ns | 1× |
| Snappy compress 1 KB | ~2 µs | 20× |
| Read 1 MB from RAM | ~10 µs | 100× |
| SSD random read | ~16 µs | 160× |
| Same-datacenter round trip | ~500 µs | 5,000× |
| Read 1 MB from SSD | ~1 ms | 10,000× |
| Disk seek (HDD) | ~5–10 ms | ~70,000× |
| Read 1 MB from HDD | ~20 ms | 200,000× |
| Round trip cross-continent | ~150 ms | 1,500,000× |

**Burn into memory:**
- RAM ≈ **100,000×** faster than a disk seek → cache.
- Same-DC round trip ≈ 0.5 ms; **cross-region ≈ 150 ms** → keep data near users, avoid chatty
  cross-region calls on the request path.
- Sequential ≫ random (why Kafka/LSM are fast).

## Data size building blocks
| Type | Size |
|------|------|
| char / ASCII | 1 B |
| int | 4 B |
| long / timestamp / double | 8 B |
| UUID | 16 B |
| Short text row / record | 0.1–1 KB |
| Tweet/post (text) | ~1 KB |
| Thumbnail | ~10–50 KB |
| Web page (HTML) | ~100 KB |
| Photo | ~0.5–2 MB |
| Minute of compressed video | ~5–50 MB |

| Power of 2 | ≈ decimal | Name |
|------------|-----------|------|
| 2^10 | ~1 thousand | KB |
| 2^20 | ~1 million | MB |
| 2^30 | ~1 billion | GB |
| 2^40 | ~1 trillion | TB |
| 2^50 | — | PB |

## Time constants (for QPS math)
- Seconds/day ≈ **86,400 ≈ 10^5** (round to 10^5 for mental math).
- Seconds/month ≈ 2.5M. Seconds/year ≈ **31.5M ≈ 3×10^7**.
- **1 million events/day ≈ ~12/sec.** (The single most useful shortcut.)
- 1 billion/day ≈ ~11,600/sec ≈ **~12K/sec**.

## QPS & peak
- QPS_avg = events_per_day ÷ 86,400.
- **QPS_peak = QPS_avg × peak_factor.** Peak factor: 2× (smooth) to 10× (spiky). State it.
- Always compute the **read:write ratio** — it dictates the architecture.

## Storage projection
`storage = record_size × records_per_day × retention_days` (× replication factor).
Project to 1 / 3 / 5 years. Separate **text/metadata** from **media/blobs** (media dominates →
blob store + CDN).

## Availability (the nines)
| Availability | Downtime/year |
|--------------|---------------|
| 99% | ~3.65 days |
| 99.9% | ~8.8 hours |
| 99.99% | ~52 min |
| 99.999% | ~5 min |

- Series deps **multiply**: 3 services @ 99.9% ≈ 99.7% end-to-end.
- Parallel redundancy improves: two 99% in parallel ≈ 99.99%.

## Single-machine rules of thumb (rough, modern commodity server)
- **RAM**: tens to hundreds of GB. A Redis node comfortably holds tens of GB of hot data.
- **A single well-tuned SQL primary**: ~thousands of writes/s, ~tens of thousands of reads/s
  (more with replicas/caching). Beyond that → cache + replicas, then shard.
- **One server holds** ~10K–100K open WebSocket connections (tune-dependent).
- **Network**: 10–100 Gbps NICs; bandwidth = QPS × payload — check it for media-heavy systems.

## Quick sanity checks
- "Need 50,000 servers"? You probably slipped a 1000× — recompute.
- Always restate assumptions ("assume 1 KB/record") so errors are visible and correctable.
