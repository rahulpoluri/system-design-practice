# URL Shortener — Phase 2

Builds on [Phase 1](phase-1.md). Only what changes or is added is written here.

## 1. Requirements

**Functional (new)**
- Sign up and sign in
- Create a short URL as a signed-in user, with a custom expiry date (optional)
- See all URLs I created
- Delete (cancel) my URLs
- Analytics per URL: clicks per day and per region, over a date range
- Anonymous create + redirect still work as in Phase 1

**Non-functional**
- Same as Phase 1: scale · 99.99% · p99 < 100 ms (same continent) · eventual consistency
- Redirect latency must NOT get worse because of analytics
- Analytics can be a few minutes behind (eventually consistent counts)
- Users can only see and delete their own URLs

**Out of scope (Phase 2)**
- Unique visitors · referrer / device analytics · SSO · teams / sharing

## 2. Back-of-the-envelope — traffic & scale
| | Calculation | Result |
|---|---|---|
| Click events (peak) | = redirect rate | ≈ 39 / s |
| Login / signup | small vs redirects | negligible |
| Analytics reads | dashboard views, rare | negligible |

- [ ] TODO: number of users (assumption needed).

## 3. API

### Sign up
- [ ] TODO (see doc fixes).

### Sign in
`POST /api/v1/login` — form body `{email, password}`

| Status | When | Response |
|---|---|---|
| `200 OK` | credentials correct | `{"access_token": "<jwt>", "token_type": "bearer"}` |
| `401 Unauthorized` | wrong email or password | `{"detail": "Username or password is incorrect"}` |
| `422` | missing fields | `{"detail": "..."}` |
| `429` | too many attempts (brute-force protection) | `{"detail": "Too many requests"}` |
| `500` / `503` | bug / backend down | as Phase 1 |

### Create a short URL
`POST /api/v1/urls` — header `Authorization: Bearer <jwt>` (optional: anonymous still allowed)

```json
{"long_url": "https://example.com/...", "expires_at": "2027-12-31T00:00:00Z"}
```

| Status | When | Response |
|---|---|---|
| `201 Created` | link created | `{"short_url": "..."}` + `Location` header |
| `401` | token sent but invalid / expired | `{"detail": "Invalid token"}` |
| `422` | invalid URL or `expires_at` in the past | `{"detail": "..."}` |
| `429` / `500` / `503` | as Phase 1 | |

### Redirect
`GET /{short_code}` — same as Phase 1, plus:

| Status | When |
|---|---|
| `410 Gone` | expired **or deleted** (`deleted_at` set) |

Side effect: emits a click event to the queue (see deep dives).

### Analytics
`GET /api/v1/urls/{short_code}/analytics?from=2026-12-22&to=2026-12-23` — header `Authorization: Bearer <jwt>`

```json
{
  "short_url": "...",
  "long_url": "...",
  "created_at": "2026-12-01T10:00:00Z",
  "clicks": [{"day": "2026-12-22", "region": "EU", "clicks": 42}]
}
```

| Status | When |
|---|---|
| `200 OK` | owner asks for their link |
| `401` | no / invalid / expired token |
| `403` | link belongs to someone else |
| `404` | code doesn't exist |
| `500` / `503` | as Phase 1 |

### List my URLs · Delete my URL
- [ ] TODO (see doc fixes).

## 4. Data model — PostgreSQL

**`users`**
| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `BIGINT` | PK (`BIGSERIAL`) | |
| `email` | `VARCHAR(256)` | `UNIQUE NOT NULL` | login name |
| `password_hash` | `VARCHAR(256)` | `NOT NULL` | bcrypt/argon2 — never the plain password |
| `joined_at` | `TIMESTAMPTZ` | `NOT NULL` | |

**`urls`** (Phase 1 table + new columns)
| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `BIGINT` | PK (`BIGSERIAL`) | unchanged — `short_code = encode(id)` |
| `long_url` | `TEXT` | `NOT NULL`, ≤ 1,024 | unchanged |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | **new** — shown in analytics |
| `expires_at` | `TIMESTAMPTZ` | `NOT NULL`, **indexed** | now user-chosen → see note below |
| `deleted_at` | `TIMESTAMPTZ` | `NULL` | **new** — soft delete |
| `user_id` | `BIGINT` | FK → `users.id`, `NULL`, **indexed** | **new** — `NULL` = anonymous; index for "list my URLs" |

**Why `expires_at` now needs an index:** in Phase 1 every link lived exactly 5 years, so the lowest ids expired first and cleanup deleted by id range. With custom expiry that's no longer true → cleanup must find rows by `expires_at`.

**`url_clicks_daily`**
| Column | Type | Constraints | Notes |
|---|---|---|---|
| `url_id` | `BIGINT` | FK → `urls.id` | |
| `day` | `DATE` | | |
| `region` | `VARCHAR(8)` | | e.g. `EU`, `IN` |
| `clicks` | `BIGINT` | `NOT NULL DEFAULT 0` | |

Composite PK `(url_id, day, region)` — also the index for "clicks of this URL between dates".

## 5. Back-of-the-envelope — storage
- [ ] TODO: `users` size · `url_clicks_daily` rows = (urls clicked per day × regions) × days kept · row size ~40 B.

## 6. High-level diagram
- [ ] TODO: Phase 1 diagram + auth, click queue, analytics workers.

## 7. Deep dives

### Analytics — every read becomes a write
- Don't write to Postgres inside the redirect: 39 writes/s on the hot path, and redirect latency would depend on the DB.
- Redirect → emit click event to a **queue** (fire-and-forget) → **worker** aggregates in memory → every few seconds upsert:
  `INSERT … ON CONFLICT (url_id, day, region) DO UPDATE SET clicks = url_clicks_daily.clicks + EXCLUDED.clicks`
- Batching = few DB writes, and counts are a few seconds/minutes behind (accepted in NFRs).
- No CDN in front of redirects (Phase 1 decision) → every click reaches us → nothing is invisible.
- [ ] Region: from GeoIP lookup on the client IP.
- [ ] Which queue? (Redis Streams / RabbitMQ / Kafka) and why.

### Auth
- JWT = **who you are (authN)**, not **what you can access (authZ)**.
- Ownership check on every "my URL" endpoint: `url.user_id == current_user.id`, else `403`. Without it → IDOR: anyone can read anyone's analytics by guessing codes (codes are sequential!).

### Delete
- Soft delete (`deleted_at = now()`) → redirect returns `410`.
- **Evict the cache** on delete, else the deleted link keeps redirecting from Redis.

## 8. NFR walk-back
- **Redirect latency:** queue publish is fire-and-forget (~1 ms) → p99 unchanged.
- **Queue down:** redirect still works; clicks are lost or buffered locally. Analytics is less critical than redirects.
- [ ] TODO: availability of auth (login down ≠ redirects down).

## 9. Doc fixes (what's wrong or missing in this file)
- [ ] **Signup** endpoint (or SSO instead?)
- [ ] **`DELETE /api/v1/urls/{code}`** → status codes table (`204` / `401` / `403` / `404`)
- [ ] **`GET /api/v1/urls`** → list my URLs, paginated (cursor or page/limit?)
- [ ] Rate limiter design behind the `429`s (e.g. token bucket in Redis, per IP / per user)
- [ ] Path naming: Phase 1 uses `/api/v1/short-urls`, Phase 2 `/api/v1/urls` — pick one
- [ ] Back-of-the-envelope: users count + `url_clicks_daily` storage
- [ ] Diagram

## 10. Self-check (answer out loud without looking — tick when I can)
- [ ] Why does analytics turn reads into writes, and why not write to Postgres in the redirect?
- [ ] Walk the click path: redirect → queue → worker → upsert. What does `ON CONFLICT` do?
- [ ] What happens to redirects if the queue is down?
- [ ] AuthN vs authZ — what is IDOR, and why is it worse with sequential codes?
- [ ] Why soft delete, and why must delete evict the cache?
- [ ] Why does `expires_at` need an index now when Phase 1 didn't?
- [ ] Why no CDN — and how does that help analytics?
