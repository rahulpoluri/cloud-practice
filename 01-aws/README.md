# AWS — services map

AWS has 200+ services; a backend developer deploying an app uses about 8 (★). The rest: know they exist.

## A URL shortener on AWS
```
DNS → Route 53 · Nginx pair → ELB · app servers → EC2 in an Auto Scaling Group (or ECS/Fargate)
Redis → ElastiCache · Postgres → RDS · Kafka → MSK · Cassandra → Keyspaces
everything inside one VPC · permissions via IAM · TLS via ACM · alerts via CloudWatch → SNS
```

## By category
| Category | What it's for | Services (★ = core) |
|---|---|---|
| Security | who can do what | ★ **IAM**, KMS, Secrets Manager |
| Network | where boxes live, who reaches them | ★ **VPC** (subnets, security groups), Route 53, CloudFront |
| Compute | running code | ★ **EC2**, ★ **ECS/Fargate**, EKS (managed K8s), Lambda |
| Traffic | spreading load, scaling | ★ **ELB**, ★ **Auto Scaling Group** |
| Data | storing things | ★ **RDS**, ★ **ElastiCache**, ★ **S3**, DynamoDB, Keyspaces |
| Ops | watching, messaging | CloudWatch, SQS/SNS, MSK (Kafka) |

## Security services
| Service | What | Example |
|---|---|---|
| ★ **IAM** | users, roles, policies — who can call which AWS API → [IAM](iam.md) | app server's role reads the DB secret, nothing else |
| ★ **Security groups** | firewall per instance (stateful, allow-only) | DB accepts 5432 only from app servers |
| NACLs | firewall per subnet (stateless, allow + deny) | rarely touched |
| ★ **Secrets Manager** / SSM Parameter Store | secrets out of code | Postgres password, JWT key |
| ★ **ACM** | free auto-renewed TLS certificates | HTTPS on the load balancer |
| KMS | encryption keys | RDS / S3 encryption at rest |
| WAF | web firewall: IP block, rate limit, SQL injection | `429` at the edge |
| Shield | DDoS protection (basic = free) | protects the load balancer |
| Cognito | managed sign-up/login for **your app's** users | alternative to own JWT auth |
| GuardDuty / Inspector / Security Hub | threats, vulnerability scans, overview | security-team tools |
| **CloudTrail** | audit log of every AWS API call | "who deleted the DB?" |

## Monitoring services
| Service | What | Example |
|---|---|---|
| ★ **CloudWatch Metrics** | CPU, requests, latency | autoscaling metric, p99 |
| ★ **CloudWatch Logs** | app + system logs | FastAPI logs |
| ★ **CloudWatch Alarms → SNS** | threshold → alert | p99 > 100 ms for 5 min |
| CloudWatch Dashboards | graphs | redirects/s, cache hit ratio |
| X-Ray | distributed tracing | app → Redis → Postgres timing |
| Managed Prometheus / Grafana | open-source stack, run by AWS | if the team uses Prometheus |
| AWS Config | history of resource settings + rule checks | "was this bucket ever public?" |

**Don't mix up:** CloudWatch = your app's health · CloudTrail = who did what in the AWS account.

## Cognito vs IAM
| | IAM users | Your app's users |
|---|---|---|
| Who | engineers, CI/CD, servers | people signing up on your website |
| Controls | access to **AWS itself** | access to **your app** |
| Stored in | IAM | your own `users` table, or **Cognito** |

Cognito user pools: sign-up/login, hashed passwords, reset, MFA, social/SSO login, issues JWTs (ID / access / refresh). Can put groups/roles in the token — but ownership checks ("is this link yours?") stay in app code. Alternatives: Auth0, Keycloak, Clerk.

## Cloud-native
Built *for* the cloud: containers · stateless services · managed services for commodity parts (DB ops, auth, queues) · elastic scaling · infrastructure as code + CI/CD · designed for failure + observability. You still write the business logic.
> "Buy the commodity, build the differentiator."
