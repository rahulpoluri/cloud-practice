# Cloud Practice

Hands-on basics for deploying a backend: AWS → Terraform → Kubernetes, one half-day block each.
Method: concept by talking (explain back) → do it by hand → short notes here. Courses = reference only.
Cost guard: free tier · `terminate` / `destroy` at the end of every session.

| # | Topic | Done = | Status |
|---|---|---|---|
| 01 | [AWS](01-aws/README.md) | EC2 + Nginx by hand, reachable in the browser, then terminated | ⬜ starts 10 Nov 16:00 |
| 02 | Terraform | same EC2 from code (docker provider first), `destroy` + recreate in one command | ⬜ |
| 03 | Kubernetes | FastAPI image on kind, 3 replicas, service on localhost | ⬜ |

Next: these feed one project — FastAPI + RAG service → Docker → AWS via Terraform → K8s → observability.
