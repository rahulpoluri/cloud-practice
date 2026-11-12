# EC2 — virtual servers

## Six launch choices
| Choice | What | Example |
|---|---|---|
| **AMI** | the disk image: OS + preinstalled software | Amazon Linux 2023, Ubuntu 24.04 |
| **Instance type** | CPU / RAM size | `t3.micro` |
| **Key pair** | SSH login key | `my-key.pem` |
| **Network** | VPC, subnet, security group, public IP yes/no | public subnet `10.0.1.0/24`, `web-sg` |
| **Storage** | EBS root volume size/type | 8 GB gp3 |
| **User data** | script run on first boot | install Nginx |

## AMI
- Per **region** (copy it to use elsewhere).
- **Custom ("baked") AMI** = your app + dependencies preinstalled → new instances start in seconds instead of installing for minutes. Matters for fast autoscaling. Built with Packer or EC2 Image Builder.

## Instance types — `m7g.large`
```
m        7            g                 large
family   generation   extra (g = Graviton/ARM, i = Intel, a = AMD, d = local disk)   size
```
| Family | For |
|---|---|
| **t** | burstable, cheap, small/dev (`t3.micro` = free tier) |
| **m** | general purpose |
| **c** | compute heavy (CPU) |
| **r** | memory heavy (caches, DBs) |
| **g / p** | GPU (graphics / ML training) |

## ARM vs x86
Two different **instruction sets** — a binary compiled for one doesn't run on the other.
- x86 = Intel/AMD. ARM = Graviton on AWS, Apple Silicon Macs.
- Graviton ≈ 20% cheaper for the same performance → default for new workloads if the stack supports it (Python, Java, Go, Node: yes).
- Docker trap: image built on an M-series Mac = arm64 → on an x86 server: `exec format error`. Fix: `docker build --platform linux/amd64 …`, or a **multi-arch image** (one tag, both builds inside; Docker pulls the matching one):
```bash
docker buildx build --platform linux/amd64,linux/arm64 -t myrepo/myapp:1.0 --push .
```

## Key pairs & SSH
- AWS keeps the public key; you download the private `.pem` **once** — lost = no SSH.
```bash
chmod 400 my-key.pem                       # ssh refuses keys others can read
ssh -i my-key.pem ec2-user@18.195.7.88     # ubuntu@ for Ubuntu AMIs
```
- SG inbound: port 22 from **your IP only** (`89.14.22.7/32`), never `0.0.0.0/0`. Home IPs change → yesterday's rule may stop matching.
- Modern alternative: **SSM Session Manager** — shell from the console/CLI, no port 22, no key, access controlled by IAM.

## EBS — the disk
Network drive attached to the instance (not a disk inside the machine).
| Action | Instance | Root EBS volume | Bill |
|---|---|---|---|
| **Stop** | off, can start again | kept | EBS (~$0.08/GB-month gp3) + Elastic IP if attached |
| **Terminate** | gone forever | deleted (default "Delete on termination") | nothing |
- Normal public IP changes on stop/start; private IP stays.
- Snapshots = backups of a volume, stored in S3 (paid per GB).
- ⚠️ Console trap: **Stop** sits right next to **Terminate** → after "terminating", check the state really says `terminated`.
- Root volume: Delete on termination = **true** by default. **Extra** volumes you add: **false** by default → they survive and keep billing.
- After terminate, check nothing bills: volumes · Elastic IPs (idle = charged hourly) · snapshots · NAT Gateways. SG + key pair are free.

## Nginx by hand (Amazon Linux 2023)
```bash
sudo dnf install -y nginx            # dnf = package manager (apt on Ubuntu)
sudo systemctl enable --now nginx    # start now + on every boot
systemctl status nginx               # active (running)?
curl localhost                       # test from inside the box
ss -tlnp                             # what is listening on which port
```
Then `http://<public-IP>` (http, no cert). SSH (22) = for you to get in and set up; 80 = for visitors.
⚠️ Don't restrict the SG **outbound** to your own IP — the instance then can't reach package repos (`dnf install` hangs). Outbound "all → 0.0.0.0/0" is normal; inbound does the filtering.

## User data
Script that runs **once, as root, on first boot**.
```bash
#!/bin/bash
dnf install -y nginx
systemctl enable --now nginx
```
Output and errors → `/var/log/cloud-init-output.log`. Install takes a minute or two → "connection refused" right after launch = Nginx not up yet (or the script failed).
