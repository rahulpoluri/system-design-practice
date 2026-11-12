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
