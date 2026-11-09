# URL Shortener — Phase 3

Builds on [Phase 2](phase-2.md). Same 7 endpoints, 100× reads / 10× everything else.

## 1. Requirements
**Functional:** same as Phase 2 (anonymous create + redirect still work).

**Scale**
- 1,000,000,000 redirects / month (peak 10×)
- 10,000,000 creates / month (peak 3×)
- Signup 100,000 · Login 1,000,000 · Delete 100,000 · List 1,000,000 · Analytics 1,000,000 / month

**Non-functional:** same as Phase 2.

## 2. Back-of-the-envelope — traffic & scale
```
All the other endpoints have ≤ 1,000,000 requests / month
Reads (login + list + analytics): 3 × 1,000,000 × 10 peak = 30,000,000
Writes (signup + delete): 2 × 100,000 × 3 peak = 600,000

Total writes => (10,000,000 × 3 + 600,000) / 2,592,000 = 30,600,000 / 2,592,000 ≈ 12 writes/s
Total reads  => (1,000,000,000 × 10 + 30,000,000) / 2,592,000 = 10,030,000,000 / 2,592,000 ≈ 3,870 reads/s
```
Read-heavy ~300:1. Code length: 600M rows over 5 years → 64⁵ = 1.07B ≥ 600M → **still 5 chars** (tight; ~10 years → 6).

## 5. Back-of-the-envelope — storage
```
Rule: index entry = key copy + 24B (one per index, not per column)

User table (100,000 × 12 × 5 = 6,000,000 rows)
Per user = 528 + 312 = 840B → 840 × 6,000,000 ≈ 5GB

Url table (10,000,000 × 12 × 5 = 600,000,000 rows)
Per url = 1060 + 92 = 1152B → 1152 × 600,000,000 ≈ 690GB

Url clicks daily (worst case: 1,000,000,000 × 12 × 5 = 60,000,000,000 rows)
Per row = 24 + 40 = 64B → 64 × 60,000,000,000 ≈ 3.8TB

All tables ≈ 4.5TB
Disk = (4,535 × 1.3 + 50) × 2 ≈ 12TB per DB node
```
Interview line: "~4.5 TB over 5 years, ~12 TB per node — too big for one Postgres node, and clicks are 85% of it."

## 7. Deep dives

### Shrink before you shard (9 Nov)
1. **Move clicks out of Postgres** → append-only analytics store (ClickHouse / Cassandra) + roll old days up into monthly totals → Postgres ≈ 690GB data → **~2TB per node**.
2. **Then:** does `urls` need sharding? 2TB on one node is manageable, writes are 12/s, cache absorbs most of 3,870 reads/s → **not yet**, it's the next step as data grows.

## Open (continue 10 Nov)
- [ ] Shard key for `urls` if we do shard (id? hash? range?) · consistent hashing
- [ ] ID generation without one central counter (range allocation per server)
- [ ] Cache at 3,870 reads/s: how many Redis nodes, hit ratio
- [ ] Multi-region?
- [ ] Phase 3 diagram
