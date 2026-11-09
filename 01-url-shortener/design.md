# 01 — URL Shortener

## 1. Requirements

**Functional**
- **Phase 1:** create a short URL, redirect to the long URL. Fixed-length codes. Retention 5 years. No sign-in, no analytics.
- **Phase 2:** sign in · cancel (delete) own URLs · list all URLs I created · analytics per URL (clicks per day, per region).
- **Phase 3:** 1 billion requests / month.

**Non-functional**
- Availability **99.99%**
- Latency **p99 < 100 ms**
- **Eventual consistency** is acceptable

**Load (Phase 1):** 1M writes/month (peak 3×), 10M reads/month (peak 10×), traffic not evenly distributed.

## 2. Back-of-the-envelope

| | Calculation | Result |
|---|---|---|
| Writes (peak) | 1M × 3 / 2,592,000 s | ≈ 1.2 / s |
| Reads (peak) | 10M × 10 / 2,592,000 s | ≈ 39 / s |
| Rows over 5 years | 1M × 12 × 5 | 60M |
| Code length | 64⁴ = 16.7M < 60M · 64⁵ = 1.07B ≥ 60M | **5 chars** (counter starts at 64⁴ so every code is exactly 5) |
| Row size | id 8 B + long_url ~1,024 B + expires_at 8 B | ≈ 1,040 B |
| Data | 60M × 1,040 B | ≈ 62 GB |
| PK index | 60M × ~32 B | ≈ 1.9 GB → fits in RAM |
| Disk | data + index + WAL/bloat headroom | ~200 GB |

`short_code` is **not stored** — it is computed from `id` (encode) and back (decode).

## 3. API

**`GET /{short_code}`** → `302` + `Location: <long_url>` · `404` not found · `410` expired/deleted · `500` · `503`

**`POST /api/v1/urls`** body `{long_url, expires_at}` → `201 {short_url}` · `422` validation · `429` rate limited · `500` · `503`

**`POST /api/v1/login`** form `{email, password}` → `200 {access_token: <jwt>, token_type: "bearer"}` · `401` wrong credentials · `422` · `429` · `500` · `503`

**`GET /api/v1/urls/{short_code}/analytics?from=&to=`** header `Authorization: Bearer <jwt>` → `200`:
```json
{
  "short_url": "...", "long_url": "...", "created_at": "...",
  "clicks": [{"day": "2026-12-22", "region": "EU", "clicks": 42}]
}
```
`401` invalid/expired token · `404` · `500` · `503`

## 4. Data model — PostgreSQL

**users**: `id BIGINT PK` · `email VARCHAR UNIQUE` · `password_hash` · `joined_at TIMESTAMP`

**urls**: `id BIGINT PK (BIGSERIAL)` · `long_url TEXT` · `created_at` · `expires_at DATE` (indexed, for cleanup) · `deleted_at TIMESTAMP NULL` (soft delete) · `user_id BIGINT FK NULL` (indexed, for "list my URLs")

**url_clicks_daily**: `url_id FK` · `day DATE` · `region` · `clicks BIGINT` — composite PK `(url_id, day, region)`, which also serves as the index for lookups by `url_id`.

**Why Postgres:** access is pure key-value by PK, no joins · load is tiny (one primary handles it) · need an atomic counter (`BIGSERIAL`/`nextval()`) · mature replication + failover (WAL streaming, Patroni). NoSQL (DynamoDB/Cassandra) is the option at Phase 3 scale.

## 5. High-level diagram

`diagram.png` — *TODO: export from Excalidraw (source → `diagram.excalidraw`).*

Client → (CDN) → LB pair (VRRP) → stateless API servers (nginx round-robin) → Redis Cluster → Postgres primary + sync standby + async read replicas (Patroni + 3-node etcd). Cleanup cron on API servers.

## 6. Deep dives

### Key generation — counter + base-64, not hashing
Cutting any hash (MD5/SHA-256) to 5 chars causes collisions → needs check + retry. A DB counter converted to base-64 is unique by design.
(Not the `base64` library: it encodes bytes, pads with `=`, and can't give exactly 5 chars.)

```python
A = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_"
POS = {c: i for i, c in enumerate(A)}

def encode(n: int) -> str:
    if n == 0:
        return A[0]
    chars = []
    while n:
        n, r = divmod(n, 64)
        chars.append(A[r])
    return "".join(reversed(chars))

def decode(s: str) -> int:
    n = 0
    for ch in s:
        n = n * 64 + POS[ch]   # KeyError → 404
    return n
```
Caveat: sequential ids → guessable codes. Fix if needed: reversible scramble (e.g. Sqids).

### Request flow
- **Create:** `INSERT … RETURNING id` → `encode(id)` → write-through to cache → return short URL
- **Redirect:** `decode(code)` → cache → miss → read replica → miss → **primary** → still missing → 404
  (primary fallback = read-your-writes despite replication lag)

### Caching
Cache is stateful → route by key, not round-robin. `hash(key) % N` breaks when N changes (most keys move → miss storm on DB).
→ **Redis Cluster**: 16,384 hash slots, the client library routes each key; adding a node moves only some slots.
Cache-aside: lazy refill on miss. A cache node down = misses, not data loss.

### High availability
- **Load balancers:** active/passive pair sharing a virtual IP (VRRP heartbeat). Scale up → active-active + DNS round-robin.
- **API servers:** stateless → round-robin.
- **DB:** 1 primary + 1 synchronous standby + async read replicas, WAL streaming. Patroni on each DB node + separate 3-node etcd (majority vote, leader lease) → prevents split-brain. Failover ≈ 10–30 s write pause.
  Sequence may skip values after failover → gaps OK, never duplicates.

## 7. NFR walk-back

**Availability 99.99% (≈ 52 min/year)**
- 1 year = 525,600 min × 0.0001 ≈ 52.6 min. Each extra 9 = 10× less downtime (99.9% ≈ 8.8 h · 99.999% ≈ 5 min).
- **Series** (LB → API → DB): availabilities multiply; shortcut: add downtime % (99.99% + 99.95% → ≈ 99.94%).
- **Parallel** (redundant copies): failure chances multiply (two × 99.9% → 99.9999%).
- Availability = MTBF / (MTBF + MTTR) → fast repair (auto-failover) matters as much as rare failure.
- Every layer is redundant with automatic failover → a single failure costs seconds. I still promise only 99.99% because detection time, **bad deploys** (most real outages) and region/DNS outages eat the budget.
- To go higher: canary deploys + auto-rollback (biggest win) · faster health checks · multi-region · graceful degradation (DB down → keep serving redirects from cache; only "create" fails).
- Known limit: single region = no protection against a full DC outage.

**Latency p99 < 100 ms — latency budget**
| Hop | Time |
|---|---|
| User → DC (same continent) | 20–50 ms |
| LB + API code | 1–5 ms |
| Redis hit | ~1 ms |
| Postgres PK lookup (index in RAM) | 2–10 ms |

Cache hit ≈ 25–55 ms · miss ≈ 30–65 ms → fits. Breaks for other continents (Mumbai → Frankfurt ≈ 150 ms network alone).
Fixes: CDN caches the redirect response at the edge (catch: first click on a new link still slow) · read-only region in Asia (replica + Redis) — works *because* we accepted eventual consistency.

**Eventual consistency**
Read replicas may lag briefly. Redirect falls back to the primary on a miss → a just-created link never 404s because of lag.

## 8. Trade-offs & open gaps (as of 2026-11-09)

**Phase 2 — still missing**
- [ ] **Analytics pipeline:** how clicks reach `url_clicks_daily` (redirect → click event to queue → worker aggregates → `INSERT … ON CONFLICT DO UPDATE clicks = clicks + n`). Region from GeoIP / CDN header.
- [ ] **CDN vs analytics conflict:** redirects cached at the CDN never reach our servers → clicks invisible. Options: `302` + `Cache-Control: no-store`, CDN logs, or edge worker.
- [ ] **Endpoints:** signup · `DELETE /api/v1/urls/{code}` (soft delete + **evict cache** + `204`) · `GET /api/v1/urls` (pagination).
- [ ] **AuthZ:** JWT = who you are, not what you can access → ownership check (`url.user_id == user.id`) else `403`.
- [ ] **Rate limiter** design behind the `429`s (e.g. token bucket in Redis, per IP / per user).
- [ ] Updated diagram (auth, queue, workers, cleanup job).

**Phase 3 (1B requests/month) — not started**
- [ ] Re-do numbers · sharding · ID generation without one central counter (range allocation per server) · NoSQL option.

**Small**
- [ ] Explain eventual consistency in my own words.
- [ ] Back-of-the-envelope calculators: practise.
- [ ] Index-size estimate and `hash % N` reasoning were marked "??" — confirm I can explain both.
