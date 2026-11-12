# Jira-like multi-tenant SaaS — Phase 2: data model

## Schema (my draft, 12 Nov evening)
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
