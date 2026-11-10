# Cloud Practice — knowledge base

What I know about running backend services in the cloud: concepts, how things fit together, commands, and pitfalls I hit hands-on.
Written in my own words, organised by topic.

## AWS
- [Services map](01-aws/README.md) — the ~8 services a backend actually uses, security + monitoring services, a URL shortener mapped to AWS
- [IAM](01-aws/iam.md) — identities, policies, how AWS decides allow/deny, CLI sign-in, least privilege for automation and AI agents

## Kubernetes vs AWS
Kubernetes is open source and runs anywhere. On AWS: **ECS/Fargate** (AWS's own, simpler) or **EKS** (managed K8s, control plane costs ~$73/month even idle). Concepts can be learned locally with **kind** for free.
> "I know the K8s concepts and ran it locally with Helm; on AWS I'd use EKS, or ECS when the team doesn't need K8s."
