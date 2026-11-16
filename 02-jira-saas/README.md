# 02 — Jira-like multi-tenant B2B SaaS

Designed in phases, like an interviewer adding requirements one by one.

| Phase | Adds | Status |
|---|---|---|
| [Phase 1](phase-1.md) | tickets, projects, hierarchy, groups/admins, audit, comments, attachments, search; mixed tenancy (2 private bank DBs), 30-day deletion, 5-year growth | ✅ done 16 Nov (diagram redraw + explain-back open) |

**Non-functional:** availability **99.99%** (active-passive, 2 EU regions) · latency **p99 < 100 ms** EU · strong writes / read-your-writes / eventual search · GDPR EU-only

Draft: `practice.txt` · Review log: [../01-url-shortener/reviews.md](../01-url-shortener/reviews.md)
