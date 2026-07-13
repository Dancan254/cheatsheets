# Napkin Math

Reach for this to estimate capacity and sanity-check latency in a design interview.
The tables below are usable now; the worked-example sections are stubs to expand.

## Latency numbers every engineer should know (order of magnitude)

| Operation | Time |
|---|---|
| L1 cache reference | ~1 ns |
| Branch mispredict | ~3 ns |
| L2 cache reference | ~4 ns |
| Mutex lock/unlock | ~17 ns |
| Main memory reference | ~100 ns |
| Compress 1 KB (zippy) | ~2 µs |
| Read 1 MB sequentially from memory | ~3 µs |
| SSD random read | ~16 µs |
| Read 1 MB from SSD | ~50 µs |
| Round trip within same datacenter | ~500 µs |
| Read 1 MB from disk (HDD) | ~1-2 ms |
| Disk seek (HDD) | ~2-10 ms |
| Round trip CA ↔ Netherlands | ~150 ms |

Takeaways: memory is ~100× faster than SSD; SSD ~100× faster than disk seek;
cross-region network dominates everything.

## Powers of two (storage sizing)

| Power | Value | ≈ |
|---|---|---|
| 2^10 | 1,024 | 1 KB |
| 2^20 | ~1 million | 1 MB |
| 2^30 | ~1 billion | 1 GB |
| 2^40 | ~1 trillion | 1 TB |
| 2^50 | - | 1 PB |

## Time (for QPS math)

- Seconds/day ≈ **86,400 ≈ 10^5**.
- QPS = daily requests / 10^5.
- **Peak QPS ≈ 2-3× average.**

## Availability (the nines)

| Nines | Downtime/year |
|---|---|
| 99% | ~3.65 days |
| 99.9% | ~8.8 hours |
| 99.99% | ~52 minutes |
| 99.999% | ~5 minutes |

## Worked examples (stub - add over time)

### QPS from DAU

### Storage from write rate × retention

### Bandwidth from payload × QPS

### Cache memory from hot-set size

## Object size rules of thumb

| Thing | ≈ size |
|---|---|
| char (ASCII) | 1 byte |
| UUID | 16 bytes |
| Typical DB row | ~1 KB (varies) |
| Small image / thumbnail | ~100 KB |
| Web page | ~1-5 MB |

## Gotchas / things I always forget

- Estimate to **one significant figure** - the interviewer wants reasoning, not precision.
- Always compute **peak**, not just average (2-3× multiplier).
- Multiply storage by **replication factor** (×3) and **retention**.
- State assumptions out loud; a wrong-but-explicit assumption is fine.
