# Components — who does what

A cluster = **control plane** (the brain: decides) + **worker nodes** (the muscle: run pods).
Golden rule: **everyone talks only to the API server**; only the API server touches etcd.
Components don't call each other — they **watch** the API server and react (the loops).

## Story: `kubectl create deployment web --replicas=3`
1. **kubectl** → HTTP request to **kube-apiserver** (the only door; checks who you are + validates).
2. API server writes the *desired state* ("Deployment web, 3 replicas") into **etcd** (the cluster's database).
3. **kube-controller-manager** (runs many controller loops) sees: Deployment wants 3, 0 exist → creates 3 Pod objects (no node yet).
4. **kube-scheduler** sees Pods with no node → picks a node for each (enough CPU/RAM requested? spread?) → writes the choice.
5. **kubelet** on that node sees "a Pod is assigned to me" → tells the **container runtime** (containerd) to pull the image and start it.
6. kubelet reports status back to the API server → `kubectl get pods` shows `Running`.
7. **kube-proxy** sets the node's network rules so a **Service** address reaches these pods; **CoreDNS** gives it a name (`web.default.svc`).

Pod crashes → kubelet restarts the container. Node dies → controller sees 2 ≠ 3 → new Pod → scheduler → another node.

## Components vs objects (two different things!)
- **Components** = programs that run (the *staff*): apiserver, etcd, scheduler, controller-manager, kubelet, kube-proxy, containerd.
- **Objects** = records stored in etcd (the *orders written in the order book*): Deployment, ReplicaSet, Pod, Service…
  `kubectl create/get/delete` works on **objects**. You never "create" a component.

Each object type has a **controller** (a loop inside controller-manager) that reads it and writes the next object down the chain:

```
Deployment  "web, 3× nginx"     ← you write this
   │  Deployment controller  (handles versions / rollouts)
   ▼
ReplicaSet  "exactly 3 of template X"
   │  ReplicaSet controller  (keeps the count)
   ▼
Pod ×3      "run nginx"         → scheduler picks a node → kubelet runs it on containerd
```
Order book analogy: Deployment = the standing order ("3 pizzas, recipe v2"), ReplicaSet = one batch ticket per recipe version,
Pod = one dish ticket. Staff only ever read/write tickets through the front desk.

## Analogy: a restaurant
| Component | Restaurant | Job |
|---|---|---|
| kube-apiserver | front desk | only point of contact; every order goes through it |
| etcd | the order book | single source of truth of what *should* exist |
| controller-manager | floor managers | keep comparing "ordered" vs "served" and fix gaps |
| scheduler | host assigning stations | decides which kitchen (node) cooks each dish |
| kubelet | head chef at each station | makes sure its assigned dishes are actually cooking |
| container runtime | the stove | actually cooks (runs the container) |
| kube-proxy | runners | route plates (traffic) to the right table |
| CoreDNS | menu / directory | names → addresses |

## Control plane
| Component | Does |
|---|---|
| kube-apiserver | the HTTP API; front door for kubectl and every component |
| etcd | consistent, highly-available key-value store of all cluster data |
| kube-scheduler | assigns unscheduled Pods to nodes |
| kube-controller-manager | runs the controllers (Deployment, ReplicaSet, Node, Job…) — the reconcile loops |
| cloud-controller-manager | *(optional)* talks to AWS/GCP: creates load balancers, notices dead VMs |

## Every node
| Component | Does |
|---|---|
| kubelet | agent: makes the Pods assigned to its node actually run + reports health |
| container runtime | runs containers (containerd, CRI-O) |
| kube-proxy | *(optional)* network rules that implement Services |

## Addons (extras, not core)
DNS (CoreDNS — practically required) · Dashboard (web UI) · resource monitoring (metrics) · cluster-level logging.

## Seen in my kind cluster (`kubectl get pods -A`)
- `kube-system`: etcd, kube-apiserver, kube-scheduler, kube-controller-manager, kube-proxy, coredns ×2
- `kindnet` = kind's pod network plugin (CNI) · `local-path-provisioner` = kind's storage addon
- **Not in the list:** kubelet and containerd — they're plain processes on the node, not pods (kubelet is what *starts* pods).
  kind's single node runs control plane + worker roles together; in prod the control plane is separate (on EKS, AWS hides it).

## Confused me
- *"Deleted a pod, 3 came back — kubelet did that?"* — **No, the controller-manager did.** Deleting a pod removes the Pod
  *object* → the **ReplicaSet controller** sees 2 ≠ 3 → creates a **new** Pod (new name, AGE a few seconds) → scheduler → kubelet starts it.
  **kubelet** only restarts a crashed **container inside an existing pod** (same pod name, `RESTARTS` goes 0 → 1).
  Rule: *new pod name = controller replaced it · same name + RESTARTS up = kubelet restarted it.*
  Chain: Deployment → owns a **ReplicaSet** (`web-5fc9f4bf66`, keeps exactly N) → owns the Pods (`web-5fc9f4bf66-xxxxx`).
- *Rollout, seen live* — `kubectl set image deployment/web nginx=nginx:1.27` → Deployment creates a **2nd ReplicaSet**,
  new scales 0→3 while old scales 3→0 **in small steps** (defaults maxSurge 25% = 1 extra pod, maxUnavailable 25% = 0 down
  for 3 replicas → never fewer than 3 ready = zero downtime). Old ReplicaSet **kept at 0**.
  `kubectl rollout undo deployment/web` → **reuses** the old ReplicaSet (scales it back to 3), no new one. That's why it's kept.
- *"Everything goes through the API server — really?"* — **Yes.** Components never call each other. Each one **watches** the
  API server and **writes back** to it (controller writes "3 pods, no node" → scheduler watches, writes "pod → node X" →
  kubelet on X watches, starts it, writes "Running"). **Only the API server talks to etcd.**
  (Small exception: `kubectl logs` / `exec` → API server calls the kubelet directly.)
- *"Where did the nginx image come from? I never pulled it."* — `nginx` = `docker.io/library/nginx:latest` (Docker Hub).
  **containerd inside the node** pulled it, not your Mac's Docker. Check: `docker exec learn-control-plane crictl images`.
- *"Do pods have internet?"* — **Yes, open by default** (outbound internet + every other pod). Lock down later with:
  **NetworkPolicy** (pod firewall) · **ServiceAccount + RBAC** (what the pod may do in K8s) · on EKS an **IAM role per
  ServiceAccount** (IRSA / Pod Identity) for AWS access.
