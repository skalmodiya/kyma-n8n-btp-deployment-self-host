# Running n8n on SAP BTP Kyma Runtime – End-to-End Beginner Guide

## Introduction

I wanted to learn **SAP BTP Kyma Runtime** by building something real instead of only reading Kubernetes concepts.

The goal was simple:

> Deploy a self-hosted instance of [n8n](https://n8n.io) on SAP BTP Kyma Runtime and expose it publicly.

During the journey I learned:

- Kubernetes Deployments
- Pods
- Services
- APIRules
- Istio sidecar injection
- Kyma Gateway
- Namespace management
- Public routing
- Troubleshooting frontend routes

At the end we successfully deployed:

```
SAP BTP Subaccount
        ↓
Kyma Runtime
        ↓
Kubernetes Cluster
        ↓
Namespace
        ↓
Deployment
        ↓
Pod
 ├── n8n container
 └── istio sidecar
        ↓
Service
        ↓
APIRule
        ↓
Kyma Gateway
        ↓
Public URL
```

---

## Prerequisites

Before starting, ensure you have the following in place.

### Installed locally

| Tool | Purpose |
|------|---------|
| [Docker Desktop](https://www.docker.com/products/docker-desktop/) | Container runtime |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Kubernetes CLI |
| [kubelogin](https://github.com/int128/kubelogin) | OIDC login for kubectl |

### SAP setup

```
SAP BTP Trial / Global Account
        ↓
Subaccount
        ↓
Kyma Runtime enabled
```

### Verify cluster connection

```bash
kubectl get namespaces
```

Expected output should include:

```
default
kyma-system
istio-system
kube-system
```

---

## Step 1 – Create Namespace

Create `namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: n8n-v2
```

Deploy it:

```bash
kubectl apply -f namespace.yaml
```

Enable Istio sidecar injection for the namespace:

```bash
kubectl label namespace n8n-v2 istio-injection=enabled
```

Verify the label was applied:

```bash
kubectl get ns n8n-v2 --show-labels
```

Expected output includes:

```
istio-injection=enabled
```

---

## Step 2 – Create Secret

The secret stores n8n's basic-auth credentials and injects them as environment variables into the container.

Create `secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: n8n-secret
  namespace: n8n-v2
type: Opaque
stringData:
  N8N_BASIC_AUTH_USER: admin
  N8N_BASIC_AUTH_PASSWORD: ChangeMe123
```

> **Important:** Change `ChangeMe123` to a strong password before deploying.

Deploy it:

```bash
kubectl apply -f secret.yaml
```

---

## Step 3 – Deploy n8n

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: n8n
  namespace: n8n-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: n8n
  template:
    metadata:
      labels:
        app: n8n
    spec:
      containers:
      - name: n8n
        image: n8nio/n8n:latest
        ports:
        - containerPort: 5678
        env:
        - name: N8N_BASIC_AUTH_ACTIVE
          value: "true"
        - name: N8N_HOST
          value: "n8n-v2"
        - name: N8N_PROTOCOL
          value: "https"
        - name: N8N_PORT
          value: "5678"
        - name: N8N_EDITOR_BASE_URL
          value: "/"
        - name: WEBHOOK_URL
          value: "/"
        envFrom:
        - secretRef:
            name: n8n-secret
```

Deploy it:

```bash
kubectl apply -f deployment.yaml
```

Verify the pod is running:

```bash
kubectl get pods -n n8n-v2
```

Expected output:

```
NAME                   READY   STATUS    RESTARTS   AGE
n8n-xxxxxxxxxx-xxxxx   2/2     Running   0          45s
```

### Why 2/2?

Kyma automatically injects an Istio sidecar into every pod in a labelled namespace:

```
Pod
 ├── n8n container       ← your app
 └── istio-proxy         ← injected by Kyma
```

`2/2` means both containers are running — this is expected and correct.

---

## Step 4 – Create Service

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: n8n-service
  namespace: n8n-v2
spec:
  selector:
    app: n8n
  ports:
  - port: 5678
    targetPort: 5678
  type: ClusterIP
```

Deploy it:

```bash
kubectl apply -f service.yaml
```

### Why do we need a Service?

Pods are ephemeral — they can be replaced at any time with a new IP address. A Service provides a stable internal endpoint that always routes to healthy pods.

Without Service:
```
APIRule → Pod directly ❌  (breaks when pod restarts)
```

With Service:
```
APIRule → Service → Pod ✅  (stable, always works)
```

---

## Step 5 – Expose via APIRule

Create `apirule.yaml`:

```yaml
apiVersion: gateway.kyma-project.io/v2
kind: APIRule
metadata:
  name: n8n-api
  namespace: n8n-v2
spec:
  gateway: kyma-system/kyma-gateway
  hosts:
    - n8n-v2
  service:
    name: n8n-service
    port: 5678
  rules:
    - path: /*
      methods:
        - GET
        - POST
        - PUT
        - PATCH
        - DELETE
        - OPTIONS
      noAuth: true
```

Deploy it:

```bash
kubectl apply -f apirule.yaml
```

Verify the APIRule is ready:

```bash
kubectl get apirule -n n8n-v2
```

Expected output:

```
NAME      STATUS   HOST
n8n-api   Ready    n8n-v2.<your-cluster-domain>
```

---

## Final Architecture

```
Browser
    ↓
Kyma Gateway  (Istio-based ingress)
    ↓
APIRule  (n8n-api — routes /* to n8n-service)
    ↓
Service  (n8n-service — ClusterIP :5678)
    ↓
Deployment  (n8n — 1 replica)
    ↓
Pod
 ├── n8n container
 └── istio-proxy
```

---

## Public URL

Kyma automatically resolves the short hostname in the APIRule to a full public URL using the cluster domain.

Given:

```yaml
hosts:
  - n8n-v2
```

Kyma exposes:

```
https://n8n-v2.<cluster-domain>
```

For example:

```
https://n8n-v2.f9cc102.stage.kyma.ondemand.com
```

No cluster ID needs to be hardcoded anywhere in your manifests.

Get your URL at any time:

```bash
kubectl get apirule n8n-api -n n8n-v2 -o jsonpath='{.spec.hosts[0]}'
```

Open the URL in your browser and log in with the credentials from `secret.yaml`.

---

## Troubleshooting — Lessons Learned

### 1. Istio injection missing

**Symptom:** Pod shows `1/1` instead of `2/2`, or traffic is blocked.

**Cause:** Namespace was not labelled for Istio injection before the pod was created.

**Fix:**

```bash
kubectl label namespace n8n-v2 istio-injection=enabled
kubectl rollout restart deployment/n8n -n n8n-v2
```

---

### 2. Frontend JS assets returned 404

**Symptom:** n8n loads a blank page or throws 404 errors for `/assets/`, `/rest/`, `/workflow/`.

**Cause:** The initial APIRule used `path: /` which only matched the root path, not sub-paths.

**Bad:**
```yaml
path: /
```

**Fix:**
```yaml
path: /*
```

The wildcard `/*` is required because n8n loads its frontend assets, API calls, and workflow routes from multiple sub-paths.

---

### 3. Hardcoding the cluster domain

**Symptom:** n8n generates incorrect webhook URLs or breaks after cluster maintenance changes the domain.

**Bad:**
```yaml
N8N_HOST: n8n.f9cc102.stage.kyma.ondemand.com
```

**Better:**
```yaml
N8N_HOST: n8n-v2
```

Kyma resolves the full domain automatically. Hardcoding the cluster ID makes the config brittle.

---

## Current Limitation

This setup uses SQLite inside the pod as n8n's default database:

```
n8n pod
   └── SQLite (inside container filesystem)
```

**Risk:** If the pod is deleted or restarted, workflow data may be lost.

**Recommended production setup:**

```
n8n pod
   ↓
Persistent Volume Claim
   ↓
PostgreSQL (external or in-cluster)
   ↓
Regular backups
```

---

## Conclusion

This project turned out to be a great hands-on introduction to Kyma Runtime.

Instead of learning Kubernetes theory in isolation, deploying n8n made the following concepts concrete and real:

| Concept | What it did here |
|---------|-----------------|
| Namespace | Isolated all n8n resources |
| Deployment | Ran the n8n container |
| Pod | Hosted n8n + Istio sidecar |
| Service | Provided stable internal routing |
| APIRule | Exposed the app to the internet |
| Kyma Gateway | Terminated TLS and routed traffic |
| Istio | Injected sidecar, enforced policies |

Final result:

```
SAP BTP
    ↓
Kyma Runtime
    ↓
Self-hosted n8n
    ↓
Public URL
```

And that was my first real application running on SAP BTP Kyma Runtime 🚀
