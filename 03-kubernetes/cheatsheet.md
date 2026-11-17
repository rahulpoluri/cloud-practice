# Kubernetes cheatsheet

## kubectl grammar — every command is the same sentence
```
kubectl  <verb>   <type>        <name>     [flags]
kubectl  get      pods                     -w
kubectl  delete   pod           web-xxx
kubectl  create   deployment    web        --image=nginx --replicas=3
kubectl  set image deployment/web  nginx=nginx:1.27      (type/name = same as "type name")
kubectl  rollout  undo          deployment/web
```
- **verb**: get (list) · describe (details + events) · create · delete · apply (from YAML) · set · rollout · logs · exec
- **type**: pod(s)/po · deployment(s)/deploy · replicaset(s)/rs · service(s)/svc · node(s)/no — singular, plural, short all work
- **flags**: `-A` all namespaces · `-n kube-system` one namespace · `-w` watch live · `-o wide` more columns (IP, node) · `-o yaml` full object
- `set image deployment/web nginx=nginx:1.27` → `nginx` before `=` is the **container name** inside the pod (= image name by default
  with `create deployment`), after `=` is the new **image:tag**. No tag = `:latest`.
- `get deploy,rs,pods` = several types at once, comma-separated, no spaces.

## kind (local cluster)
```bash
brew install kind
kind create cluster --name learn     # ~1 min first time (downloads node image)
kind get clusters
kind delete cluster --name learn     # throw everything away
```

## kubectl — where am I?
```bash
kubectl config current-context       # which cluster kubectl talks to (kind-learn)
kubectl get nodes                    # the machines
kubectl get pods -A                  # all pods, all namespaces (-A) — incl. K8s's own components in kube-system
```

## Deployments, ReplicaSets, rollouts
```bash
kubectl create deployment web --image=nginx --replicas=3
kubectl get deploy,rs,pods -o wide      # the whole chain + which node/IP
kubectl get pods -w                      # -w = watch live (Ctrl+C to stop)
kubectl delete pod <name>                # controller replaces it (new name)
kubectl set image deployment/web nginx=nginx:1.27   # rolling update → new ReplicaSet
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web      # back to previous ReplicaSet
```
