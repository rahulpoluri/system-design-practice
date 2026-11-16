# Functional requirements
- different companies use it and they have to see just their data
- we should be able to read tickets write tickets and edit tickets
- create user profiles with multiple authorization levels
- can be accessed only by login
- each person has their tickets and state of each tickets
- there are branching tickets with heirarchy with different types relating to each other and different projects 
- it should have log trail for every state change
- we should be able to search and filter results

# non functional requirements
- it can be eventual consistency
- it should be highly available
- data integrity is of utmost importance(should hold same state everywhere)
- share infra with some customer needing seperated databases
- remove all their data within 30 days
- total 3*10^5 users and 10^5 active everyday
- 50reads/day/user 5writes/day/user searches 5/day/user
- current 50*10^6 tickets and 10^7 growing every year
- each tenents  10-100 users avg tops 5000 users

growth in 5 years
-------------------
- Tenants go from 2,000 -> 15,000
- users 300,000 -> 2*10^6
- dau  10^5 -> 7*10^5
- tickets 50* 10^6 -> 250 *10^6


# BOE for traffic
reads = 7* 10^5 * 50 / 12 / 60 / 60 * 10(for spikes) ~ 8000 requests/sec
writes = 7 * 10^5 * 5 /12/60/60 * 5(for spikes) ~ 400 requests/sec

writes = WAL fsync + every index + locks → one primary; 
reads = RAM + snapshots → scale out with replicas.


# Doc fixes (open — start of session 2)
- [ ] Consistency contradiction: replace "eventual consistency" + "data integrity" with the split → STRONG for ticket writes, status changes, audit log · EVENTUAL for search, dashboards, notifications
- [ ] Add NFR: all data stays in the EU (GDPR)
- [ ] Name the separate-DB tenants: 2 bank tenants (contract) — everyone else shared
- [ ] Capacity check (my answers, own words): 400 writes/s → one Postgres primary OK · 8,000 reads/s → read replicas + cache (cache keys prefixed with tenant_id; ticket page read from primary = read-your-writes) · search → Elasticsearch/OpenSearch fed async from Postgres, every query filtered by tenant_id
- [ ] Rule-of-thumb table: Postgres ~1–5k writes/s · ~10k+ reads/s · Redis ~100k ops/s
- [ ] Insight to say out loud: scale is small → the hard part is TENANCY (isolation, bank DBs, noisy neighbours, 30-day deletion)
- [ ] Clarifying questions I asked (tenants, compliance, traffic, 5-yr growth) → list them at the top

# Next (session 2)
Diagram first: user → ? → DB. Where does tenant_id come from on each request? Tenancy model (shared table + tenant_id / schema / DB per tenant) → mixed.

# Data model (12 Nov evening)

## Schema
```
tenet_table
------------
tened_id: Integer: 8B
tenet_name: VARCHAR: 20B
is_shared: BOOL: 1B
db_name: VARCHAR: 20B

users
----
id: INTEGER: 8B PK
tenet_id INTEGER: 8B FK
name: VARCHAR : 20B
position: VARCHAR: 20B
permissions_json: JSON : 1K
group_id : INTEGER: 8B FK

index on tenet_id, group_id
index on name

groups
------
id: INTEGER: 8B PK
tenet_id INTEGER: 8B FK
group_name: VARCHAR : 20B
permissions_json: JSON : 1K

composite index on tenet_id, group_name

tickets
-------
id: INTEGER: 8B PK
tenet_id INTEGER: 8B FK
author: INTEGER: 8B FK
title: VARCHAR : 256B
type: VARCHAR: 20B
description: VARCHAR: 1KB
parent_to: INTEGER: 8B
child_to: INTEGER: 8B
created_at: DATETIME: 8B
updated_at: DATETIME: 8B

composite index on tenet_id, created_at
index on author

comments
---------
id: INTEGER: 8B PK
tenet_id INTEGER: 8B FK
ticket_id: INTEGER: 8B FK
user_id: INTEGER: 8B FK
timestamp: DATETIME: 8B
content: VARCHAR: 1KB
links_path: JSON: 32B

composite index on tenet_id, ticket_id, timestamp  # comments of a ticket in order
composite index on tenet_id, user_id   # "all comments by user X" within a tenant
# (plain tenet_id index dropped: covered by the composite's leftmost prefix)

logs
----
id: INTEGER: 8B PK
tenet_id INTEGER: 8B FK
ticket_id: INTEGER: 8B FK
timestamp: DATETIME: 8B
event: VARCHAR: 20B

index on tenet_id, timestamp
index on ticket_id


we will have multiple db with same structure if some one wants isolated
db they get isolated db with different db name but same table structure
```

## Review — fixed in session
- ✅ `comments` was a 10 KB JSON column in `tickets` → own table, one row per comment.
  Why: adding to JSON = read whole blob → append → write back. Two people at the same second → **lost update** (second write wipes the first). 500 comments → every new comment rewrites the whole row (Postgres writes a new row version on every UPDATE). No author, can't edit/delete one comment.
  Analogy: one shared notebook everyone takes home and rewrites vs. everyone sticks their own sticky note.
- ✅ `comments` missing `tenant_id` → added (rule: tenant_id on EVERY table).
- ✅ `tickets` had no timestamp → can't do "newest first" → `created_at` + `(tenant_id, created_at)`.
- ✅ Mixed tenancy: `is_shared` + `db_name` in tenant table = routing to an isolated DB. Say this out loud.

## Open gaps (session 2)
- [ ] `users.group_id` = one group per user → real life is many-to-many → `user_groups (user_id, group_id)` join table.
- [ ] `parent_to` + `child_to` → only `parent_id` needed (children = `WHERE parent_id = X`).
- [ ] tickets missing: `project_id` (+ projects table), `status`, `assignee_id`, `priority`.
- [x] (12 Nov, 21:18) Indexes reworked to composites per query; spotted the redundant plain `tenant_id` index on comments myself. Was: too many single-column indexes → keep only those a real query needs (each index slows every write). Drop plain `tenant_id` indexes where a composite starting with `tenant_id` exists.
- [ ] Typo: tenet → tenant.

# Storage + RAM sizing (15 Nov)

## Storage (5 years: 250M tickets · 3 comments/ticket · 10 state changes/ticket)
Rule: row size + index entries (each index entry ≈ key + ~24 B) → × rows.

| Table | Rows | Row + indexes | Total |
|---|---|---|---|
| tickets | 250M | 1,356 + 104 = 1,460 B | **~365 GB** |
| comments | 750M | 1,096 + 120 = 1,216 B | **~910 GB** |
| logs | 2.5B | 52 + 104 = 156 B | **~390 GB** |
| users + groups + tenants | 2.4M | — | ~3 GB → ignore |

- **Total ≈ 1.7 TB** → × 1.3 (bloat + WAL) → × 2 headroom ≈ **4.5 TB SSD per DB node**.
- Comments dominate (1 KB text is conservative; real average is probably ~300 B).
- Logs: tiny rows but 3 indexes → **indexes 2× the data**. Keep only indexes a real query uses; partition by month, old months → cold storage.
- Interview: size only the BIG tables, say "users/tenants are tiny, ignore".
- Still fits one Postgres primary → the hard part is TENANCY, not size.

## RAM sizing — the rule
**RAM = (indexes, or their HOT part if too big) + recent data pages + a little for WAL & connections → × 2.**

- **Small DB** (indexes fit, e.g. URL shortener ~3 GB): put ALL indexes in RAM. Done.
- **Big DB** (D2 indexes ≈ 375 GB): don't fit everything. Fit what is READ OFTEN = the **hot set / working set**.
- RAM = speed, SSD = capacity. Page in RAM ≈ 0.1 ms · from SSD ≈ 1 ms+. Nobody puts all data in RAM.

**What must be in RAM (D2 minimum):**
| Part | Why | Size |
|---|---|---|
| Upper levels of every B-tree (root + inner nodes, ~1% of index) | every lookup walks through them | ~4 GB |
| Hot set: last month's tickets + comments + logs + their index leaf pages (250M / 60 months ≈ 4M tickets/month) | where the 8k reads/s land | ~25 GB |
| WAL buffers + connections (~10 MB each) + sort memory + OS | runs the server | ~3 GB |
| **Absolute minimum** | below this hot pages evict each other → SSD reads → p99 breaks | **~32 GB** |
| **Provision (× 2)** | growth + spikes | **64 GB** |

**No info at all?** RAM ≈ 10% of total data (D2: ~170 GB) — safe, generous.

**Say it:** *"SSD for everything, RAM for what's hot, × 2. Data is 1.7 TB, indexes ~375 GB, so neither fits in RAM. The working set is last month's tickets, about 30 GB, so minimum 32 and I'd provision 64."*

## Concepts I mixed up
### Composite index — not only "WHERE a = x AND b = y"
My idea: a composite index is for `WHERE id = … AND name = …`. True, but that example is the one where it's useless — `id` is unique, `name` adds nothing.

**Phone book sorted by (surname, first name):**
| Query | Uses the index? |
|---|---|
| surname = Kumar AND first name = Rahul | ✅ filter + filter |
| surname = Kumar | ✅ **leftmost prefix** — all Kumars are together |
| surname = Kumar ORDER BY first name | ✅ filter + sort — already in order, no sort step |
| first name = Rahul only | ❌ Rahuls are scattered |

- Rule: **filter columns first, sort column last.** `(ticket_id, timestamp)` = comments of one ticket in order · `(tenant_id, created_at)` = a tenant's newest tickets.
- A query usually uses **one** index → build the index to match the whole query, not one per column.
- `(tenant_id, id)` doesn't speed up lookup by `id` (PK already finds one row) — `WHERE tenant_id = …` is still there for **safety**, not speed. But it can **replace** a plain `tenant_id` index (leftmost prefix) → one index instead of two → cheaper writes.

# Diagrams + data flow (16 Nov)

## Diagrams
- Overview (first ugly draft): `overview-v1.png`
- Write flow: `write-flow.png`
- Read flow: `read-flow.png`

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
