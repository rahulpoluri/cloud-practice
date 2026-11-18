# mongo + mongo-express on kind

Browser → mongo-express-service → mongo-express pod → (ConfigMap: `mongo-service`) → mongo-service (ClusterIP) → mongodb pod

Apply order: `secret.yaml` → `mongo.yaml` (Deployment + Service) → `configmap.yaml` → `mongo-express.yaml` (Deployment + Service)

Open: `kubectl port-forward service/mongo-express-service 8081:8081` → http://localhost:8081 (web login admin/pass)

Secret = `root/root`, local learning cluster only. Real systems: never commit plain Secrets → External Secrets Operator + AWS Secrets Manager / Sealed Secrets / SOPS, plus RBAC + etcd encryption.

## Pitfalls hit
- `data:` in a Secret must be base64 (`echo -n root | base64`; `-n` = no trailing newline).
- Renaming `metadata.name` + apply = a SECOND object, not a rename → delete the old one.
- Read error paths: `containers[0].env[1].valueFrom.key` = 2nd env var missing the `secretKeyRef:` level.
- ConfigMap ref = `configMapKeyRef`, Secret ref = `secretKeyRef` (no plain `keyRef`).
- mongo-express 1.0 entrypoint pings `ME_CONFIG_MONGODB_URL` (default `mongo`) → "Name does not resolve" retries are noise; the app uses `ME_CONFIG_MONGODB_SERVER`.
- kind: NodePort isn't reachable from the Mac without extraPortMappings → use `kubectl port-forward`.
- Debug: `kubectl logs deployment/<name>`, `kubectl exec deployment/<name> -- env`, `kubectl describe pod` (Events).
