# FastAPI + Postgres on Kubernetes — how real teams run a service

```
Browser → Ingress → Service → FastAPI pods (2–5, HPA)
                                   ↓
                         Service → Postgres (StatefulSet + PVC)
Deployed by: own Helm chart ← ArgoCD ← GitHub repo
```

| Layer | What it gives you |
|---|---|
| Postgres via Helm (e.g. Bitnami chart) | Third-party software comes from charts; you only override values |
| App in a Docker image | Same artifact everywhere; config comes from env vars |
| Deployment + Service + Secret + probes | K8s restarts a hung pod (liveness), sends traffic only when ready (readiness) |
| Ingress + HPA | Reachable from outside; scales pods under load |
| Own Helm chart | Same app in dev and prod, only values differ |
| ArgoCD | Push to Git → cluster syncs itself (GitOps) |
| AWS + Terraform | Runs in the cloud; DB swaps to RDS by changing only DATABASE_URL |

## Why run Postgres on K8s at all
- The skill is "install someone else's chart" — same for Redis, Keycloak, Prometheus, ingress-nginx.
- Dev/test DBs commonly run on K8s; production is usually managed (RDS/Cloud SQL).
- Sovereign/bare-metal clouds (no RDS) run Postgres on K8s via an operator (CloudNativePG).
- The app only knows DATABASE_URL → local K8s Postgres or RDS, the app code doesn't change.
