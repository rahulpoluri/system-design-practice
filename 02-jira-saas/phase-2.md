# D2 Jira SaaS — Phase 2: Diagrams + data flow

## Diagrams
- Overview (first ugly draft): `phase-2-overview-v1.png`
- Write flow: `phase-2-write-flow.png`
- Read flow: `phase-2-read-flow.png`

Boxes: DNS · LB (active/passive) · app ×2 · Redis ×2 · event queue → indexer ×2 → Elasticsearch ·
notifier queue → notifier ×2 · shared DB bank + private DB bank (primary / standby / 2 read replicas each) · S3-compatible object storage.

## Tenant routing
- `tenant_id` comes from the auth token (JWT claim), not from a lookup.
- **Tenant directory** (tenant_id → DB bank) lives in ONE place (shared DB / control plane).
- Every app server holds the whole directory **in memory** (10k tenants × ~100 B ≈ 1 MB) → 0 network hops.

## Write flow
1. Client → LB → app.
2. App → **primary** of the tenant's bank: UPDATE ticket + INSERT audit row **in the same transaction** (sync, no extra hardware, one commit).
3. App → cache: **DELETE** the key (or SET with version — see below).
4. App → event queue (search indexing) + notifier queue (email / webhooks / @mentions). Async.
5. Response → client.

Audit log ≠ event queue: audit = permanent legal record, must never be lost → same txn. Queue = transport, lost messages can be rebuilt.

## Read flow (cache-aside)
1. App → cache. Hit → return.
2. Miss → app → **read replica** directly (cache and DB never talk to each other).
3. App → cache SET with TTL.
4. Full-text search → Elasticsearch.

## Cache consistency
- **Race:** two writers commit v7 then v8, but their cache SETs arrive v8 then v7 → stale cache. DB orders DB writes, nothing orders cache writes.
- **Option A — DELETE on write:** a delete can't write a wrong value; next read refills.
- **Option B — SET + version:** `version` column on cached + updated tables (tickets, projects, users; not append-only tables). Cache accepts only a higher version (Redis Lua compare-and-set). Use the DB version, never timestamps (clock skew).
- **Replica lag:** after a write, a read from a lagging replica can return AND cache the old value → **read-your-writes**: store `last_write_at` in session, route that user's reads to the primary for ~1–2 s.
- Names: eventual consistency for others · read-your-writes for the author · CAS on the cache. Strong consistency (linearizability) = everything from primary = no read scaling.
- Bonus: the version column = **optimistic locking** (`UPDATE ... WHERE id=42 AND version=7`; 0 rows → someone else edited).

### DECISION (16 Nov) — two consistent packages, don't mix them
| | A: DELETE on write | B: SET with version (**chosen for tickets**) |
|---|---|---|
| Write | UPDATE → DELETE cache key | partial `UPDATE ... SET status=..., version=version+1 RETURNING *` → cache CAS-SET (only if version higher) |
| Author sees own write | needs read-your-writes → author's reads go to **primary** ~1–2 s (else replica refills cache with old row) | instantly from cache, **no primary routing** |
| Extra column | none | `version` on tickets only |
| Two users edit different fields at once | partial UPDATEs → Postgres row lock orders them, both survive | same + `RETURNING *` gives the full merged row, higher version wins in cache |
- Replica lag in B only matters on a cache miss after TTL → by then the replica caught up (TTL ≫ lag).
- Never read-modify-write the whole row in the app → lost update. Partial UPDATE, or optimistic lock if a whole-row form is saved.
- Other ways to version without a column: Postgres `xmin` (change detection only, wraps around) · CDC/Debezium (one ordered reader, WAL LSN = version, no app writes to cache).

## Attachments (S3 / MinIO)
- Never stream files through app servers (double bandwidth, worker blocked for minutes, LB timeouts).
- Endpoints:
  1. `POST /tickets/42/attachments {filename,size,type}` → `201 {attachment_id, upload_url, expires_in: 300}` (row = `pending`)
  2. `PUT <upload_url>` — client → S3 directly
  3. `POST /attachments/7/complete` → `uploaded` (optional if S3 ObjectCreated event → queue → worker marks it)
- **Pre-signed URL** = app signs locally (HMAC-SHA256 with its secret over method + bucket/path + expiry). No AWS call, no registration. S3 recomputes the signature from the `X-Amz-*` query params; any change → 403. Can never exceed the signer's IAM permissions.
- Cleanup cron: `pending` > 24 h → HeadObject; exists → mark uploaded, else delete row.
- S3 = replicated across ≥3 AZs (one box in diagram). On-prem: MinIO / Ceph (erasure coding, ≥4 nodes, same S3 API → only `endpoint_url` changes).

## Gaps still open
- [ ] Draw tenant router + auth service + arrows to the private bank.
- [ ] One event → two queues = app publishes twice (dual write). Consider one topic + 2 consumer groups / outbox pattern.
- [ ] Notifier → client: emails go via an email provider (SES), in-app via websocket — not straight back to the client.
- [ ] API + tenancy deep dive.
