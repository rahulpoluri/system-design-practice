# URL Shortener — Phase 2

## 1. Requirements (added on top of Phase 1)
- Sign in
- Cancel (delete) my URLs
- See all URLs I created
- Analytics per URL: number of clicks, which region, which day, average

## 2. Back-of-the-envelope
- [ ] TODO: users table size · `url_clicks_daily` rows (urls × days × regions with clicks) · click-event rate (= read rate, ~39/s peak)

## 3. API

**`GET /{short_code}`** → `302` + `Location` · `404 {"detail": "link not found"}` · `410` expired **or deleted** · `500` · `503`

**`POST /api/v1/urls`** body `{"long_url": "...", "expires_at": "<date>"}` (custom expiry) → `201 {"short_url": "..."}` · `422` · `429` · `500` · `503`

**`POST /api/v1/login`** form `{email, password}` → `200 {"access_token": "<jwt>", "token_type": "bearer"}` · `401 {"detail": "Username or password is incorrect"}` · `422` · `429` · `500` · `503`

**`GET /api/v1/urls/{short_code}/analytics?from=2026-12-22&to=2026-12-23`** header `Authorization: Bearer <jwt>` → `200`:
```json
{
  "short_url": "...", "long_url": "...", "created_at": "...",
  "clicks": [{"day": "2026-12-22", "region": "EU", "clicks": 42}]
}
```
`401` invalid/expired token · `404` · `500` · `503`

## 4. Data model — PostgreSQL

**users**: `id BIGINT PK` · `email VARCHAR(256) UNIQUE` · `password_hash VARCHAR(256)` · `joined_at TIMESTAMP`

**urls**: `id BIGINT PK` · `long_url TEXT` · `created_at TIMESTAMP` · `expiry_date DATE` (indexed, for cleanup) · `deleted_at TIMESTAMP NULL` (soft delete) · `user_id BIGINT FK NULL` (indexed, for "list my URLs")

**url_clicks_daily**: `url_id BIGINT FK` · `day DATE` · `region` · `clicks BIGINT` — composite PK `(url_id, day, region)`, which also serves as the index for lookups by `url_id`.

## 5. High-level diagram
- [ ] TODO: Phase 1 diagram + auth, click queue, analytics workers.

## 6. Deep dives

### Analytics (discussed 6 Nov — write up properly)
- Key insight: **analytics turns every read into a write.** Don't write to Postgres inside the redirect.
- Redirect → emit click event to a **queue** → **worker** aggregates → upsert `INSERT … ON CONFLICT (url_id, day, region) DO UPDATE SET clicks = clicks + n`.
- [ ] Region: from GeoIP / CDN header.
- [ ] **CDN vs analytics conflict:** redirects cached at the CDN never reach us → invisible clicks. Options: `302` + `Cache-Control: no-store`, CDN logs, edge worker.

### Auth
- JWT = **who you are (authN)**, not **what you can access (authZ)**.
- [ ] Ownership check on every URL endpoint: `url.user_id == current_user.id`, else `403` — otherwise IDOR (anyone can read anyone's analytics by guessing codes).

## 7. NFR walk-back
- [ ] TODO: does the queue affect redirect latency? (No: fire-and-forget.) What if the queue is down? (Redirect still works; clicks lost or buffered.)

## 8. Open gaps
- [ ] **Signup** endpoint (or SSO instead?)
- [ ] **`DELETE /api/v1/urls/{code}`** → soft delete (`deleted_at`) + **evict cache** (else deleted link keeps redirecting) + `204`
- [ ] **`GET /api/v1/urls`** → list my URLs, paginated
- [ ] Rate limiter design behind the `429`s (e.g. token bucket in Redis, per IP / per user)
- [ ] Path naming: Phase 1 used `/api/v1/short-urls`, Phase 2 `/api/v1/urls` — pick one
