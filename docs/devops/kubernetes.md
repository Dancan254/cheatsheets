# Kubernetes

Reach for this when driving `kubectl`, reading a manifest, or debugging why a pod won't come up.

## Mental model

- **Pod** → one or more containers sharing network/storage. Smallest deployable unit. Ephemeral.
- **ReplicaSet** → keeps N identical pods running. You rarely touch it directly.
- **Deployment** → manages ReplicaSets → gives you rolling updates + rollback.
- **Service** → stable virtual IP/DNS in front of a set of pods (pods die, Service stays).
- **Ingress** → HTTP(S) routing from outside the cluster to Services (host/path rules).
- **ConfigMap / Secret** → config + sensitive data injected as env vars or files.
- **Namespace** → virtual cluster for isolating resources.

## kubectl - the daily verbs

```bash
kubectl get pods                         # -A all namespaces, -o wide for node/IP
kubectl get pods -n prod -w              # watch
kubectl get deploy,svc,ingress           # multiple kinds at once
kubectl describe pod my-pod              # events at the bottom = why it's broken
kubectl logs my-pod                      # -f follow, --previous for crashed container
kubectl logs deploy/my-app               # logs from a deployment's pods
kubectl exec -it my-pod -- sh            # shell in
kubectl apply -f manifest.yaml           # declarative create/update
kubectl delete -f manifest.yaml
kubectl rollout status deploy/my-app     # is the rollout done?
kubectl rollout undo deploy/my-app       # rollback to previous ReplicaSet
kubectl rollout restart deploy/my-app    # bounce all pods (e.g. to reload a Secret)
```

## Debugging a bad pod (the flow)

```bash
kubectl get pod my-pod                   # STATUS column tells you the category
kubectl describe pod my-pod              # scroll to Events - image pull? scheduling? probe?
kubectl logs my-pod --previous           # crashed? read the last container's logs
kubectl get events --sort-by=.lastTimestamp   # cluster-wide recent events
```

STATUS → cause:

| STATUS | Usually means |
|---|---|
| `ImagePullBackOff` / `ErrImagePull` | Bad image name/tag, or missing registry credentials |
| `CrashLoopBackOff` | Container starts then exits → read `logs --previous` |
| `Pending` | Can't schedule → no node resources, or unbound PVC (`describe` says which) |
| `CreateContainerConfigError` | Missing ConfigMap/Secret referenced in the spec |
| `OOMKilled` | Hit memory limit → raise limit or fix the leak |
| `Running` but not `Ready` | Readiness probe failing |

## Deployment manifest (the shape)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels: { app: my-app }
  template:
    metadata:
      labels: { app: my-app }
    spec:
      containers:
        - name: my-app
          image: registry.example.com/my-app:1.4.2   # pinned
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: "250m", memory: "256Mi" }   # scheduler reserves this
            limits:   { cpu: "1",    memory: "512Mi" }    # killed if exceeded
          readinessProbe:                                  # gates traffic
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 10
          livenessProbe:                                   # restarts if dead
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            initialDelaySeconds: 20
          envFrom:
            - configMapRef: { name: my-app-config }
            - secretRef: { name: my-app-secrets }
```

## Service + Ingress

```yaml
apiVersion: v1
kind: Service
metadata: { name: my-app }
spec:
  selector: { app: my-app }        # matches pod labels
  ports: [{ port: 80, targetPort: 8080 }]
  # type: ClusterIP (default, in-cluster) | NodePort | LoadBalancer
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: my-app }
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: my-app, port: { number: 80 } } }
```

## Config & context

```bash
kubectl config get-contexts
kubectl config use-context prod-cluster
kubectl config set-context --current --namespace=prod   # stop typing -n prod
kubectl create configmap my-app-config --from-literal=LOG_LEVEL=INFO
kubectl create secret generic my-app-secrets --from-literal=DB_PASSWORD=secret
```

## Port-forward (test a service locally)

```bash
kubectl port-forward svc/my-app 8080:80        # localhost:8080 → service
kubectl port-forward pod/my-pod 5005:5005      # remote debug
```

## Gotchas / things I always forget

- **Liveness vs readiness:** liveness failing → pod **restarted**; readiness failing → pod **removed from Service** (no traffic) but keeps running. Mixing them up causes restart loops.
- A too-aggressive liveness probe (short `initialDelaySeconds`) restarts a slow-booting app forever → `CrashLoopBackOff` that isn't actually a crash.
- **Service selector must match pod labels exactly** or the Service has no endpoints (silent - `kubectl get endpoints my-app` shows empty).
- `requests` is what the scheduler reserves; `limits` is the hard cap. No `requests` → pods can get scheduled onto starved nodes. No memory `limit` → one pod can OOM the node.
- Editing a ConfigMap/Secret does **not** restart pods. `kubectl rollout restart deploy/...` to pick up changes.
- `kubectl delete pod` on a Deployment-managed pod just makes a new one - you want `kubectl delete deploy`.
- `apply` is declarative and idempotent; `create` fails if it already exists. Use `apply` for everything.
- Namespaces: forgetting `-n` means you're staring at `default` wondering where your pods went. Set the context namespace.

## Quick reference

| Task | Command |
|---|---|
| Why is it broken | `kubectl describe pod X` → Events |
| Crashed logs | `kubectl logs X --previous` |
| Shell in | `kubectl exec -it X -- sh` |
| Rollback | `kubectl rollout undo deploy/X` |
| Reload config | `kubectl rollout restart deploy/X` |
| Local access | `kubectl port-forward svc/X 8080:80` |
| No traffic? | `kubectl get endpoints X` (empty = selector mismatch) |
| Default ns | `kubectl config set-context --current --namespace=NS` |
