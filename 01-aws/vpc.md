# Networking basics → VPC

## Addresses
| | What | Who gives it |
|---|---|---|
| **MAC address** | fixed ID of a network card (`be:1c:95:53:06:c6`) | factory (phones/Macs may randomise per WiFi) |
| **Private IP** | address inside a local network (`192.168.0.30`) | router via **DHCP**, or set by hand (static) |
| **Public IP** | address on the internet | internet provider (home) / AWS pool (cloud; **Elastic IP** = fixed one) |

- **NAT**: the router swaps private IPs for its one public IP on the way out and remembers who asked → replies come back, but nobody outside can *start* a connection in.
- Two computers on a cable, no router → no DHCP → self-assigned `169.254.x.x` (link-local) or static IPs in the same range.
- **IP = envelope address**, stays the same to the final destination. **MAC = next hop**: every router rewrites the destination MAC to the next box; no device's own MAC ever changes.
- `127.0.0.1` = localhost (loopback, the machine talking to itself).
- macOS: `ifconfig` → `en0` = WiFi (`inet` = private IP, `ether` = MAC, `netmask 0xffffff00` = `/24`).

## CIDR — the slash
IPv4 = 4 numbers × 8 bits = 32 bits. The number after the slash = how many bits are fixed. **It is a block size, not a count.**

```
free bits = 32 − slash        addresses = 2 ^ free bits
```

| CIDR | First – last | Addresses |
|---|---|---|
| `192.168.0.0/16` | `192.168.0.0 – 192.168.255.255` | 65,536 |
| `192.168.16.0/20` | `192.168.16.0 – 192.168.31.255` | 4,096 |
| `192.168.1.0/24` | `192.168.1.0 – 192.168.1.255` | 256 |
| `192.168.0.64/26` | `192.168.0.64 – 192.168.0.127` | 64 |
| `192.168.0.30/32` | `192.168.0.30` | 1 |
| `0.0.0.0/0` | everything | all |

+1 on the slash halves the block, −1 doubles it. Blocks are powers of 2 and start at a multiple of their size: `192.168.0.0 – 192.168.15.255` ✅, `192.168.1.0 – 192.168.10.255` ❌.

Same notation at every level: `192.168.0.30/24` on a laptop = "my IP, and my local network is `192.168.0.0 – 192.168.0.255`".

**Private ranges** (only these): `10.0.0.0/8` · `172.16.0.0/12` · `192.168.0.0/16`. Anything else belongs to someone on the internet.

## Route table
"For destination X, send to Y." Most specific match wins.
```
192.168.0.0/24 → local (deliver directly)
0.0.0.0/0      → router 192.168.0.1 (everything else)
```

## VPC
**VPC** = your private network in AWS, one CIDR block, e.g. `10.0.0.0/16` (`10.0.0.0 – 10.0.255.255`). AWS allows `/16` (biggest) to `/28` (smallest) → use `/16`, it's free. Each region has a **default VPC** (`172.31.0.0/16`).

**Subnet** = a smaller block inside the VPC, in exactly **one availability zone**; must fit inside the VPC, must not overlap. AWS reserves 5 addresses per subnet.
```
VPC     10.0.0.0/16   10.0.0.0 – 10.0.255.255
 ├ 10.0.1.0/24       10.0.1.0 – 10.0.1.255   public,  AZ-a
 ├ 10.0.2.0/24       10.0.2.0 – 10.0.2.255   public,  AZ-b
 ├ 10.0.11.0/24      10.0.11.0 – 10.0.11.255 private, AZ-a
 └ 10.0.12.0/24      10.0.12.0 – 10.0.12.255 private, AZ-b
```

**Why subnets at all:** rules are set per group, not per machine — internet access (route table per subnet), placement across data centres (one subnet per AZ), and firewall rules by range (`allow 5432 from 10.0.11.0/24` covers servers autoscaling adds later).

## Public vs private subnet
There is no "public" checkbox — it's the route table:
| Subnet | `0.0.0.0/0` goes to | Effect |
|---|---|---|
| public | **Internet Gateway** (VPC's front door) | in + out |
| private | **NAT Gateway** (sits in a public subnet) | out only (pip install, Docker pull, Stripe) |
| isolated | nothing | no internet at all |

A private subnet without a route can't reach the internet no matter what the firewall allows — routing comes first, firewall second.

⚠️ NAT Gateway ≈ $0.05/h (~$35/month) even idle, not free tier → skip or delete right away when practising.

## URL shortener layout
```
Internet → Internet Gateway
  [public subnets, AZ-a + AZ-b]   load balancer (+ NAT Gateway)
  [private subnets, AZ-a + AZ-b]  app servers → Redis, Postgres
```
> "Only the load balancer is public; everything else is in private subnets across two AZs, going out through NAT."

## Route table targets (AWS)
| Target | Prefix | Used for |
|---|---|---|
| local | `local` | traffic inside the VPC — always there, can't delete |
| Internet Gateway | `igw-` | public subnet: `0.0.0.0/0 → igw` |
| NAT Gateway | `nat-` | private subnet: `0.0.0.0/0 → nat` |
| VPC peering | `pcx-` | another VPC, e.g. `10.1.0.0/16 → pcx` |
| VPN gateway | `vgw-` | office / on-prem network |
| Transit Gateway | `tgw-` | hub connecting many VPCs |
| Gateway endpoint | `vpce-` + prefix list `pl-` | S3 / DynamoDB without internet |

Longest prefix wins: `10.0.11.0/24` beats `10.0.0.0/16` beats `0.0.0.0/0`.
**Route table = the road map (can the packet get there?). Security group = the bouncer (is it allowed in?).**

## Security group vs NACL
| | Security group | NACL |
|---|---|---|
| Attached to | instance (its network card, ENI) | subnet |
| Rules | allow only | allow + deny, numbered, first match wins |
| State | **stateful** — reply traffic allowed automatically | **stateless** — must allow return traffic (ephemeral ports `1024 – 65535`) too |
| Default | deny all in, allow all out | default NACL allows everything |
| Used for | everyday firewall | blocking a bad IP range, compliance "deny" rules |

**SG can reference another SG:** `db-sg: allow 5432 from app-sg` = only instances wearing the `app-sg` badge; covers instances autoscaling adds later, no IPs needed.
⚠️ Instances in the same SG can **not** talk to each other automatically — only the VPC's **default** SG has that self-referencing rule. For a custom SG, add `allow from <itself>` explicitly.

## Debug ladder — what the error tells you
| You see | Layer | Check |
|---|---|---|
| **timeout** | network — packet dropped silently | SG inbound rule (port + your IP) → public IP? → route `0.0.0.0/0 → igw`? → NACL |
| **connection refused** | machine reached, nothing listening on that port | is the app/Nginx running? right port? |
| **HTTP 404 / 500 / 502** | the app answered | app code, config, upstream |

## Elastic IP
Fixed public IPv4 that stays when the instance stops/restarts (a normal auto-assigned public IP is released on stop and changes on start).
- Needed for: NAT Gateway, partners who whitelist your IP, DNS records pointing straight at an instance.
- Not needed behind a load balancer: the ALB has a stable **DNS name** (`my-alb-123.eu-central-1.elb.amazonaws.com`) → Route 53 **alias** record points your domain at it. Need fixed IPs for the LB → NLB with EIPs, or Global Accelerator.
- 💸 Every public IPv4 costs $0.005/h (~$3.60/month), including EIPs left idle.

## VPC endpoints — reach AWS services without internet
Many AWS services live **outside** your VPC with public endpoints, protected by IAM: S3, DynamoDB, SQS, SNS, ECR, Secrets Manager, CloudWatch. Services **inside** your VPC: EC2, RDS, ElastiCache, load balancers.

A private instance reaching S3 normally goes via NAT (paid per GB). Endpoints avoid that:
| | Gateway endpoint | Interface endpoint (PrivateLink) |
|---|---|---|
| Services | S3, DynamoDB only | most others (SQS, ECR, Secrets Manager…) |
| How | entry in the route table (`pl-… → vpce-…`) | a network card with a private IP in your subnet |
| Cost | **free** | ~$0.01/h per AZ + per GB |

Other NAT alternatives: egress-only internet gateway (IPv6 only, free), NAT instance (cheap EC2 doing NAT, you manage it).

## Costs
| Free | Paid |
|---|---|
| IAM, VPC, subnets, route tables, Internet Gateway, SGs, NACLs, gateway endpoints, inbound data | NAT Gateway (~$0.05/h + per GB), public IPv4 ($0.005/h, incl. idle EIPs), interface endpoints, data out to internet (first 100 GB/month free), **cross-AZ** traffic (~$0.01/GB each way) |

**Saving in production:** gateway endpoints for S3/DynamoDB · one shared NAT in dev (one per AZ only in prod) · keep chatty services in the same AZ · CloudFront in front for egress · instances in private subnets behind an ALB (no public IPs) · release unused EIPs · stop non-prod at night/weekends · tags + Budgets + Cost Anomaly Detection · IAM/SCP guardrails on expensive instance types.
