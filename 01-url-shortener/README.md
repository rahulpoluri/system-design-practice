# 01 — URL Shortener

Designed in phases, like an interviewer adding requirements one by one.

| Phase | Adds | Status |
|---|---|---|
| [Phase 1](phase-1.md) | create + redirect, 5-year retention, scrambled 5-char codes | ✅ |
| [Phase 2](phase-2.md) | sign-in, delete, list my URLs, click analytics | ✅ done 9 Nov (rate limiter → D5) |
| [Phase 3](phase-3.md) | 1 billion requests / month | ✅ done 10 Nov — no shard (vertical + replicas), clicks → Cassandra, Redis + Sentinel |

**Non-functional (all phases):** availability **99.99%** · latency **p99 < 100 ms** · **eventual consistency** OK · **non-enumerable codes**

Review log: [reviews.md](reviews.md)
