# Kubernetes — the big picture

## Why it exists
Docker = package an app so it runs anywhere — but on **one machine**, started by hand.
Real systems = hundreds of containers on dozens of machines. Someone must answer, all day:
- which machine has room for this container?
- a container crashed → who restarts it?
- a machine died → who moves its containers?
- traffic doubled → who starts more copies?
- new version → who swaps containers one by one, without downtime?
- copies keep moving → how does service A find service B?

Kubernetes (K8s) automates all of that.

## The one idea: desired state + reconciliation
You don't give commands ("start X on machine 3"). You **declare the state you want**
("3 copies of my-app v2, 512 MB each"). Kubernetes loops forever:

> observe what's running → compare with what was asked → fix the difference → repeat

Pod crashed (2 of 3) → starts one. Machine died → reschedules its pods. v2 → v3 → replaces gradually.
Every component either **stores** the desired state or **works to make reality match it**.

**Analogy:** a thermostat — set 21 °C, it keeps checking and correcting. K8s = thermostat for containers.
**Already known:** AWS Auto Scaling Group ("keep 3 EC2 healthy") is the same loop for VMs; K8s generalises it
to containers, networking, config, storage, rollouts.

## How we got here (physical → VM → container)
| Era | How apps ran | Problem it left |
|---|---|---|
| Physical | 1 server, many apps, no boundaries | one greedy app starves the others; 1 app/server = expensive, idle |
| VM | many VMs on one server, each with its **own full OS** | isolated + better use, but every VM carries a whole OS (heavy, slow to boot) |
| Container | apps **share the host OS kernel**, each has own filesystem/CPU/RAM share | light + portable — but now there are *hundreds* of them → need a manager → **K8s** |

Container image = immutable → same image laptop → test → prod; rollback = run the previous image.

## What it gives you (the loop applied to everything)
| Feature | In plain words |
|---|---|
| Self-healing | restart crashed containers, replace them, kill ones failing their health check, no traffic until *ready* |
| Rollouts / rollbacks | change v2 → v3 at a controlled rate; go back if it breaks (canary too) |
| Service discovery + load balancing | stable DNS name/IP in front of changing copies, traffic spread across them |
| Bin packing | you state CPU/RAM per container → it fits them onto nodes |
| Horizontal scaling | more/fewer copies by command or automatically on CPU |
| Secrets + config | passwords/config injected at runtime — no rebuild of the image |
| Storage | mounts disks (local, cloud) into containers |
| Batch | run-to-completion jobs, retried on failure |
| Extensible | add your own resource types without changing K8s itself |

## What it is NOT
- **Not a build/CI tool** — doesn't build images or deploy source code (that's GitHub Actions etc.).
- **Not a database/queue/cache provider** — Postgres, Redis, Kafka can *run on* it, but it doesn't ship them.
- **Not opinionated about logging/monitoring** — you plug in Prometheus, Grafana, etc.
- **Not "orchestration" in the strict sense** — no fixed script "do A, then B, then C". Many **independent control loops**,
  each pushing current state → desired state. No central conductor → more robust (one loop failing doesn't stop the others).

## Why it's everywhere
1. Open-source version of **Borg**, what runs Google internally.
2. **Portable** — same YAML on EKS (AWS), GKE (Google), AKS (Azure), or kind on a laptop.
3. **Ecosystem** — monitoring, CI/CD, service meshes all plug into it → it's the standard.

**Downside:** complex. For a small app it's overkill — one EC2 or ECS/Fargate is simpler. Knowing *when not to* use it shows understanding.

## Local cluster: kind
**kind = Kubernetes IN Docker.** Each "node" (machine) is a Docker container on the laptop, running a real Kubernetes inside.
Flight simulator: real controls, not a real plane — kubectl commands work the same on EKS.

| Tool | Job |
|---|---|
| **kind** | *builds* the cluster (create / delete) — rarely used |
| **kubectl** | *talks to* the cluster (run, list, delete things) — used constantly; same on any cluster |

kind = the builder; kubectl = the remote control. On AWS, EKS replaces kind, kubectl stays.

## Topics
- [Components](components.md) — control plane vs node, who does what, request lifecycle
- [Cheatsheet](cheatsheet.md) — kind + kubectl commands

## Confused me
- *"Why several copies on one machine?"* — (1) Python uses ~1 core per process (GIL) → more processes use more cores,
  (2) one crashes, others serve, (3) rolling update without a gap, (4) different apps share leftover CPU/RAM.
  Catch: same machine dies → all copies die → in prod K8s **spreads** copies across nodes / AZs.
- *"Python only uses one core?"* — for **pure-Python CPU work**, yes: the **GIL** lets one thread run Python bytecode at a time.
  Threads still help for **I/O waits** (GIL released while waiting) and C libs (NumPy) release it.
  Use all cores = **more processes**: `multiprocessing`, gunicorn `-w 4`, or more pods. (Optional no-GIL build since 3.13, not default.)
- *"8 cores = max 8 pods?"* — no. CPU is counted in **millicores** (1 core = 1000m). Pod **request** (e.g. `250m`) = reserved,
  used by the scheduler to place it; **limit** = cap (CPU → throttled, memory → OOMKilled). Node fills up when *requests* sum to
  capacity: 8000m / 250m ≈ 32 pods (minus system pods); hard default cap 110 pods/node. Single-process Python pod requesting
  > 1000m = waste → scale with more pods instead.
- *ECS vs EKS* — **ECS** = AWS's own, simpler container service (AWS-only); **Fargate** = don't manage machines, pay per container;
  **EKS** = real Kubernetes on AWS (portable, more complex, ~$73/month control plane even idle).
- *"Didn't we install kubectl?"* — kubectl was already there (comes with Docker Desktop). kind ≠ kubectl: one builds, one talks.

## Service (preview — hands-on next)
- Pods get a **new IP** every time they're replaced → nobody can call a pod by IP.
- **Service** = stable name + IP in front of pods, finds them by **label** (`app=web`), load-balances. Others call `http://web`
  (CoreDNS → Service IP → kube-proxy → a live pod). Analogy: call hotel **reception**, not room 214.
- Types: **ClusterIP** (default, inside cluster) · **NodePort** (port on every node, testing) · **LoadBalancer** (cloud LB, public).
- **Deployment file = what to run + how many** (replicas + pod template with labels). **Service file = how to reach them**
  (`selector` = same label, `port` → `targetPort`). They never reference each other — **the label is the link**.
  `kubectl apply -f deployment.yaml -f service.yaml`
