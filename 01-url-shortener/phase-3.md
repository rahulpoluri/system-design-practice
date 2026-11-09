# URL Shortener — Phase 3

## 1. Requirements
- **1 billion requests / month**

## Open gaps (not started)
- [ ] Re-do back-of-the-envelope (≈ 385 req/s avg; peak?)
- [ ] Sharding: by what key? Consistent hashing
- [ ] ID generation without one central counter (range allocation per server)
- [ ] When NoSQL (DynamoDB/Cassandra) beats Postgres
- [ ] Revisit 99.99%: multi-region?
