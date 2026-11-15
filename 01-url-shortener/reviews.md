# Reviews — URL shortener

Honest log: what I could do from memory, what I missed.

| Date | Type | Result |
|---|---|---|
| 2026-10-26 | Design session | HLD v3: counter + base-64 keys, Postgres, Redis Cluster, HA (VRRP, Patroni) |
| 2026-11-06 | Design session | NFR walk-back (availability math, latency budget), endpoints, Phase 1 cleanup job, Phase 2 tables |
| 2026-11-09 | Gap review | Phase 2 incomplete: analytics pipeline, signup/delete/list endpoints, authZ 403, rate limiter. Phase 3 not started |
| 2026-11-15 | **1st timed run from blank** (30-min slot) | **~60 min work (2×)** + 15 min break. ~70% recalled. Structure solid: reqs → estimates → DB → API → diagram → cleanup → NFR check. Gaps below |
| next | Timed run, DIFFERENT design, HARD 30-min stop | — |

Spaced repetition (NFRs): 7 Nov · 9 Nov · 13 Nov · 15 Nov (folded into the timed run)

## 2026-11-15 timed run — gaps
1. **Time 2×.** Wrote full sentences + exact maths. Interview mode: keywords only, say the reasoning, round (86,400 ≈ 10⁵ s/day). Encode pseudocode = deep dive, one line unless asked.
2. **301 vs 302** swapped. 301 = permanent (browser caches forever) · 302 = temporary → needed for expiry (410) + click counting. Also "509" → 503; redirect was missing 404 / 410.
3. **Cache drawn as consistent hashing (sharding).** Sharding solves size, replication solves failure. Hot set fits one node → primary + replica + Sentinel.
4. **Write DB = single point of failure.** Failover: Patroni + 3-node etcd (majority vote, leader lock → no split-brain), promote the **sync** standby, ~10–30 s write pause.
5. **Code length not calculated.** 91M rows: 64⁴ = 16.7M < 91M ≤ 64⁵ = 1.07B → 5 chars.
6. **Availability had no number** at first → 99.99% ≈ 52 min/year.
7. **Latency check skipped the network hop** (user → DC 20–50 ms = most of the 100 ms budget).
8. Cleanup lock name: `pg_try_advisory_lock` (auto-released on crash → no state table / Redis TTL needed).
9. Index entry ≈ key + ~24 B ≈ 32 B (not 8 B) → 91M × 32 B ≈ 2.9 GB, still fits RAM.

