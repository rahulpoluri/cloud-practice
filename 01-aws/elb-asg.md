# Load balancer + Auto Scaling

## Why
One instance = single point of failure, and 3 instances = 3 IPs. The load balancer gives **one stable address** and only sends traffic to healthy instances.
```
User → ALB (public subnets, 2 AZs) → instances (private subnets, no public IP)
```

## Types
| | Layer | For |
|---|---|---|
| **ALB** (Application) | 7 — HTTP/HTTPS | web apps, APIs, path/host routing |
| **NLB** (Network) | 4 — TCP/UDP | extreme throughput, fixed IPs (EIPs), non-HTTP |
| GWLB | 3 | firewall appliances — rare |

## ALB parts
| Part | What | Example |
|---|---|---|
| **Listener** | port + protocol the ALB accepts | 443 HTTPS (TLS cert from ACM), 80 → redirect to 443 |
| **Listener rules** | which target group gets the request | `/api/*` → api-tg · `shop.example.com` → shop-tg · 90% v1-tg / 10% v2-tg (canary) |
| **Target group** | a "team" of backends doing the same job: members + port + health check | api-tg: 3 instances on port 80 |
| **Health check** | pings each member, stops traffic to failures | `GET /health` every 30 s, healthy on 200 |

Within a target group: **round robin** (default) or **least outstanding requests**. Sticky sessions (cookie keeps a user on one instance) exist — avoid; they mean state on the server.

## Security groups
```
alb-sg: allow 80/443 from 0.0.0.0/0
app-sg: allow 80 from alb-sg        ← only the ALB can reach the instances
```
**One instance unhealthy → that instance's app** (Nginx crashed, user data failed). **All unhealthy → shared config** (SG, health check path, port).

## Auto Scaling Group (ASG)
Keeps the right number of instances running.
| Setting | Meaning |
|---|---|
| **min** | never fewer |
| **desired** | what runs right now |
| **max** | never more |

- **Launch template** = the recipe (AMI, instance type, SG, user data, IAM role) the ASG launches from.
- **Self-healing:** instance fails health check or is terminated → ASG launches a replacement to get back to desired.
- **Scaling policies:** target tracking ("keep average CPU at 50%"), step scaling, scheduled (more at 09:00), predictive.
- ASG ↔ ALB: new instances are registered in the target group automatically, removed ones deregistered. Use **ELB health checks** on the ASG so an app failure (not just a dead VM) triggers replacement.

### Common mix-ups
- ❌ "ELB is basically Nginx." → Same job, but **managed**: no server to run, patch or scale; AWS spreads it across AZs.
- ❌ "ASG only repairs / keeps the minimum." → Two jobs: **self-healing** (back to desired) **and scaling** (policy changes desired with load).
- ❌ "ASG needs an AMI." → It needs a **launch template**; the AMI is one part of it (plus type, SG, user data, IAM role). User data must set the app up — nobody SSHs in at 3 a.m.
- ❌ "Scale on memory" out of the box. → EC2 reports CPU/network by default; memory needs the **CloudWatch agent**.
- ❌ "Request → load balancer → checks → target group." → **Listener → rules → target group → healthy instance.** Health checks run continuously in the background, not per request.
- ❌ Pin `app-sg` to the ALB's IP. → ALB IPs change (it scales internally) → reference **`alb-sg`**.

⚠️ Terminating ASG instances by hand doesn't stop them — the ASG brings them back. To shut down: set desired/min to 0, or delete the ASG.
