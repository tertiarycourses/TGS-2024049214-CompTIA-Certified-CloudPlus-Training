# Lab 32 — Container Orchestration with Kubernetes (k3s)

In this lab you will install **k3s** (a minimal, single-binary Kubernetes), deploy an app, expose it, scale it, and watch a rolling update — the canonical workload-orchestration platform on every cloud.

## Lab platform

Run this lab on the **Killercoda Kubernetes Playground** — it gives you a real cluster with `kubectl` already configured:

https://killercoda.com/playgrounds/scenario/kubernetes

> The Ubuntu playground has no cluster. If you prefer to work locally, enable Kubernetes in **Docker Desktop** (https://www.docker.com/products/docker-desktop/) instead.

---

## Step 1 — Install k3s

```bash
apt update && apt install -y curl
curl -sfL https://get.k3s.io | sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes
```

---

## Step 2 — Deploy an application

> Ready-made file: [`app.yaml`](app.yaml) — you can download it instead of typing this block.

```bash
# 1. Check Kubernetes configuration
kubectl config current-context
kubectl config get-contexts

# 2. Use the Kubernetes admin configuration
export KUBECONFIG=/etc/kubernetes/admin.conf

# 3. Verify connection to API server
kubectl get nodes
kubectl cluster-info

# 4. Create application manifest
cat > app.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.26-alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
EOF

# 5. Deploy
kubectl apply -f app.yaml

# 6. Wait for deployment
kubectl rollout status deployment/web

# 7. Check pods and service
kubectl get pods,svc

# 8. Test application
curl -s http://localhost:30080 | head -3

# If localhost does not work, use the node IP:
curl -s http://172.30.1.2:30080 | head -3
```

---

## Step 3 — Scale (horizontal)

```bash
kubectl scale deploy/web --replicas=5
kubectl get pods -l app=web
```

This is the **horizontal scaling** primitive from Lab 19, but managed.

---

## Step 4 — Rolling update (Lab 12 idea, K8s-native)

```bash
kubectl set image deploy/web web=nginx:1.27-alpine
kubectl rollout status deploy/web
kubectl rollout history deploy/web
```

Rollback if anything breaks:

```bash
kubectl rollout undo deploy/web
```

---

## Step 5 — Persistent volume

> Ready-made file: [`pvc.yaml`](pvc.yaml) — you can download it instead of typing this block.

```bash
cat > pvc.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data }
spec:
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 100Mi } }
EOF
kubectl apply -f pvc.yaml
kubectl get pvc
```

k3s ships with `local-path` provisioner — equivalent to AWS EBS / Azure Disk.

---

## Step 6 — Liveness probe (self-healing)

```bash
kubectl patch deploy/web --type=json -p '[
  {"op":"add","path":"/spec/template/spec/containers/0/livenessProbe",
   "value":{"httpGet":{"path":"/","port":80},"initialDelaySeconds":3,"periodSeconds":5}}
]'
kubectl get pods -l app=web
```

If a pod fails its probe, the kubelet restarts it. **Self-healing**.

---

## Step 7 — RBAC (Lab 24 idea, K8s-native)

```bash
kubectl create role pod-reader --verb=get,list --resource=pods
kubectl create rolebinding alice-pods --role=pod-reader --user=alice
kubectl auth can-i list pods --as alice
kubectl auth can-i delete pods --as alice
```

---

## Ready-made manifests

Every object above is also available as a proper, apply-ready YAML file in the
[`manifests/`](manifests/) folder — clone or download them instead of typing the
inline YAML and `kubectl create` commands:

| File | Lab step | Apply with |
|------|----------|------------|
| [`manifests/deployment.yaml`](manifests/deployment.yaml) | Step 2 — the `web` Deployment (3 × `nginx:1.26-alpine`) | `kubectl apply -f manifests/deployment.yaml` |
| [`manifests/service.yaml`](manifests/service.yaml) | Step 2 — the NodePort Service on 30080 | `kubectl apply -f manifests/service.yaml` |
| [`manifests/pvc.yaml`](manifests/pvc.yaml) | Step 5 — the `data` PersistentVolumeClaim | `kubectl apply -f manifests/pvc.yaml` |
| [`manifests/liveness.yaml`](manifests/liveness.yaml) | Step 6 — the Deployment with the liveness probe set | `kubectl apply -f manifests/liveness.yaml` |
| [`manifests/rbac.yaml`](manifests/rbac.yaml) | Step 7 — the `pod-reader` Role + `alice-pods` RoleBinding | `kubectl apply -f manifests/rbac.yaml` |

Or apply the whole set at once:

```bash
kubectl apply -f manifests/
```

See [`manifests/README.md`](manifests/README.md) for details.

---

## Step 8 — Map K8s ↔ exam objectives

| K8s primitive | CV0-004 concept |
|---------------|-----------------|
| Deployment | Workload orchestration |
| Service | Application/network load balancer |
| ConfigMap / Secret | CaC + Secrets management |
| HPA | Horizontal scaling |
| PVC | Persistent volume |
| RBAC | Authorization model |
| NetworkPolicy | Security group |

---

## Step 9 — Cleanup

```bash
kubectl delete -f app.yaml
kubectl delete pvc data
/usr/local/bin/k3s-uninstall.sh
```

---

## Test it

Run these checks to prove the lab worked before you move on:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes
kubectl get deploy,pods,svc -l app=web
kubectl rollout history deploy/web
kubectl get pvc data
kubectl auth can-i list pods --as alice
curl -s http://localhost:30080 | head -3
```

**Expected:** Run this before Step 9. The node reports status **Ready**; the `web` Deployment shows `5/5` ready pods (after the Step 3 scale) all in `Running`; `rollout history` lists at least two revisions from the `nginx:1.26-alpine` → `1.27-alpine` update; the `data` PVC is **Bound** to a `local-path` volume; `kubectl auth can-i list pods --as alice` answers **yes** (while `delete pods` answers **no**); and the NodePort on 30080 returns the nginx welcome HTML.

---

## What you learned
- k3s gives you a real cluster in one shell command.
- Deployments, services, scaling, rolling updates, RBAC.
- K8s implements every cloud-orchestration primitive on the exam.

## Free tools used
- k3s — https://k3s.io
- kubectl — bundled
- Minikube (alternative) — https://minikube.sigs.k8s.io
- k9s (terminal UI) — https://k9scli.io
- Lens Desktop (free) — https://k8slens.dev
- Killercoda K8s playground — https://killercoda.com/playgrounds/scenario/kubernetes

---

## Files in this lab

| File | Purpose |
|------|---------|
| [`app.yaml`](app.yaml) | Step 2 Deployment + NodePort Service applied to the cluster. |
| [`pvc.yaml`](pvc.yaml) | Step 5 PersistentVolumeClaim backed by the local-path provisioner. |
| [`manifests/deployment.yaml`](manifests/deployment.yaml) | Step 2 — apply-ready `web` Deployment (3 × nginx:1.26-alpine). |
| [`manifests/service.yaml`](manifests/service.yaml) | Step 2 — apply-ready NodePort Service on port 30080. |
| [`manifests/pvc.yaml`](manifests/pvc.yaml) | Step 5 — apply-ready `data` PersistentVolumeClaim (100Mi). |
| [`manifests/liveness.yaml`](manifests/liveness.yaml) | Step 6 — the `web` Deployment with the liveness probe set (self-healing). |
| [`manifests/rbac.yaml`](manifests/rbac.yaml) | Step 7 — the `pod-reader` Role and `alice-pods` RoleBinding. |
| [`manifests/README.md`](manifests/README.md) | Index of the manifests with the `kubectl apply -f` command for each. |
