# AWS basics — map (10 Nov 2026)

200+ services, a backend dev deploying an app uses ~8. ★ = learn now, the rest = know it exists.
Method: concept by talking (explain back) → do it by hand in the console → short notes here. Maarek course = reference only, not front-to-back.

## Your URL shortener on AWS
```
DNS → Route 53 · Nginx pair → ELB · app servers → EC2 in an ASG (or ECS/Fargate)
Redis → ElastiCache · Postgres → RDS · Kafka → MSK · Cassandra → Keyspaces
everything inside one VPC · permissions via IAM · TLS via ACM · alerts via CloudWatch → SNS
```

## Map by category
| Category | What it's for | Services (★ = core 8) |
|---|---|---|
| Security | who can do what | ★ **IAM**, KMS, Secrets Manager |
| Network | where boxes live, who reaches them | ★ **VPC** (subnets, security groups), Route 53, CloudFront |
| Compute | running code | ★ **EC2**, ★ **ECS/Fargate**, EKS (managed K8s, ~$73/mo idle), Lambda |
| Traffic | spreading load, scaling | ★ **ELB**, ★ **Auto Scaling Group** |
| Data | storing things | ★ **RDS**, ★ **ElastiCache**, ★ **S3**, DynamoDB, Keyspaces |
| Ops | watching, messaging | CloudWatch, SQS/SNS, MSK (Kafka) |

## Security
| Service | What | In the URL shortener |
|---|---|---|
| ★ **IAM** | users, roles, policies — who can call which AWS API | app server's role reads the DB secret, nothing else |
| ★ **Security groups** | firewall per instance (stateful, allow-only) | DB accepts 5432 only from app servers |
| NACLs | firewall per subnet (stateless, allow + deny) | rarely touched |
| ★ **Secrets Manager** / SSM Parameter Store | secrets out of code | Postgres password, JWT key |
| ★ **ACM** | free auto-renewed TLS certs | HTTPS on the LB |
| KMS | encryption keys | RDS / S3 encryption at rest |
| WAF | web firewall: IP block, rate limit, SQLi | `429` at the edge |
| Shield | DDoS protection (basic = free) | protects the LB |
| Cognito | managed sign-up/login | alternative to own JWT auth |
| GuardDuty / Inspector / Security Hub | threats, vuln scans, overview | security-team tools — know names |
| **CloudTrail** | audit log of every AWS API call | "who deleted the DB?" |

## Monitoring
| Service | What | In the URL shortener |
|---|---|---|
| ★ **CloudWatch Metrics** | CPU, requests, latency | ASG scaling metric, p99 |
| ★ **CloudWatch Logs** | app + system logs | FastAPI logs |
| ★ **CloudWatch Alarms → SNS** | threshold → alert | p99 > 100 ms for 5 min |
| CloudWatch Dashboards | graphs | redirects/s, cache hit ratio |
| X-Ray | distributed tracing | app → Redis → Postgres timing |
| Managed Prometheus / Grafana | open-source stack, AWS-run | if team uses Prometheus |
| AWS Config | resource setting history + rules | "was this bucket ever public?" |

**Don't mix up:** CloudWatch = your app's health · CloudTrail = who did what in the AWS account.

## Kubernetes vs AWS
K8s is open source, runs anywhere. AWS options: **ECS/Fargate** (AWS's own, simpler) or **EKS** (managed K8s). Learn K8s locally with **kind** (free).
Interview line: *"K8s concepts + ran it locally with Helm; on AWS I'd use EKS, or ECS if the team doesn't need K8s."*

## Half-day plan
1. **AWS** — IAM · VPC/SG · EC2 · ELB/ASG → EC2 + Nginx by hand → **terminate**
2. **Terraform** — docker provider (local) → same EC2 from code → `destroy`
3. **Kubernetes** — kind, FastAPI image, 3 replicas, service
Cost guard: free tier · destroy/terminate at the end of every session.

## Session notes
_(filled in after each hands-on block)_
