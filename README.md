# n8n on SAP Kyma / BTP — Self-Hosted Deployment

Deploy [n8n](https://n8n.io) — the open-source workflow automation platform — as a self-hosted instance on **SAP Business Technology Platform (BTP)** using **SAP Kyma** (managed Kubernetes).

This repository contains the complete set of Kubernetes manifests needed to get n8n running behind the Kyma API Gateway with basic-auth protection, in under 10 minutes.

---

## Table of Contents

1. [What is n8n?](#what-is-n8n)
2. [Architecture Overview](#architecture-overview)
3. [Repository Structure](#repository-structure)
4. [Prerequisites](#prerequisites)
5. [Step-by-Step Deployment Guide](#step-by-step-deployment-guide)
   - [Step 1 — Clone the Repository](#step-1--clone-the-repository)
   - [Step 2 — Update Your Credentials](#step-2--update-your-credentials)
   - [Step 3 — Connect kubectl to Your Kyma Cluster](#step-3--connect-kubectl-to-your-kyma-cluster)
   - [Step 4 — Create the Namespace](#step-4--create-the-namespace)
   - [Step 5 — Apply the Secret](#step-5--apply-the-secret)
   - [Step 6 — Deploy n8n](#step-6--deploy-n8n)
   - [Step 7 — Expose the Service](#step-7--expose-the-service)
   - [Step 8 — Create the API Rule](#step-8--create-the-api-rule)
   - [Step 9 — Verify the Deployment](#step-9--verify-the-deployment)
   - [Step 10 — Access n8n](#step-10--access-n8n)
6. [Configuration Reference](#configuration-reference)
7. [Updating n8n](#updating-n8n)
8. [Teardown / Cleanup](#teardown--cleanup)
9. [Troubleshooting](#troubleshooting)
10. [Security Considerations](#security-considerations)

---

## What is n8n?

[n8n](https://n8n.io) is a **fair-code, self-hostable workflow automation tool** — similar to Zapier or Make (Integromat) — that lets you connect apps, automate tasks, and build complex integrations through a visual node-based editor. Because it is self-hosted, your data never leaves your own infrastructure.

---

## Architecture Overview

```
Internet
    │
    ▼
┌──────────────────────────────────────────────┐
│  SAP Kyma (managed Kubernetes on BTP)        │
│                                              │
│  ┌─────────────────┐                         │
│  │  Kyma Gateway   │  (Istio-based ingress)  │
│  └────────┬────────┘                         │
│           │  APIRule: n8n-v2 (all HTTP verbs)│
│           ▼                                  │
│  ┌─────────────────┐  Namespace: n8n-v2      │
│  │  Service        │  ClusterIP :5678        │
│  │  n8n-service    │                         │
│  └────────┬────────┘                         │
│           │                                  │
│           ▼                                  │
│  ┌─────────────────┐                         │
│  │  Deployment     │  n8nio/n8n:latest       │
│  │  n8n (1 pod)    │  Basic-auth enabled     │
│  └─────────────────┘                         │
│           │                                  │
│  Secret: n8n-secret (credentials)            │
└──────────────────────────────────────────────┘
```

| Component | Kind | Purpose |
|-----------|------|---------|
| `namespace.yaml` | `Namespace` | Isolated namespace `n8n-v2` |
| `secret.yaml` | `Secret` | Basic-auth credentials injected as env vars |
| `deployment.yaml` | `Deployment` | Runs the n8n container |
| `service.yaml` | `Service` | Internal ClusterIP service on port 5678 |
| `apirule.yaml` | `APIRule` | Kyma gateway rule that exposes the service externally |

---

## Repository Structure

```
kyma-n8n-btp-deployment-self-host/
├── namespace.yaml      # Kubernetes Namespace (n8n-v2)
├── secret.yaml         # Opaque Secret with n8n credentials
├── deployment.yaml     # n8n Deployment (1 replica)
├── service.yaml        # ClusterIP Service on port 5678
└── apirule.yaml        # Kyma APIRule for external access
```

---

## Prerequisites

Before you begin, make sure you have the following:

### Tools

| Tool | Minimum Version | Install Guide |
|------|----------------|---------------|
| `kubectl` | v1.26+ | [kubernetes.io/docs](https://kubernetes.io/docs/tasks/tools/) |
| `git` | any | [git-scm.com](https://git-scm.com/downloads) |

### SAP BTP / Kyma

- An active **SAP BTP subaccount** with the **Kyma Environment** enabled
- The **kubeconfig** file for your Kyma cluster downloaded from the BTP Cockpit
- **Kyma API Gateway** module installed on the cluster (enabled by default on most Kyma instances)

### Permissions

Your BTP user needs at least the **Namespace Admin** cluster role (or **Kyma admin**) to create namespaces and apply APIRules.

---

## Step-by-Step Deployment Guide

### Step 1 — Clone the Repository

```bash
git clone https://github.com/skalmodiya/kyma-n8n-btp-deployment-self-host.git
cd kyma-n8n-btp-deployment-self-host
```

---

### Step 2 — Update Your Credentials

> **Important:** The default password in `secret.yaml` is a placeholder. Change it before deploying to any environment.

Open `secret.yaml` in your editor:

```yaml
stringData:
  N8N_BASIC_AUTH_USER: admin          # <-- change to your desired username
  N8N_BASIC_AUTH_PASSWORD: ChangeMe123  # <-- change to a strong password
```

Save the file. These values are injected directly as environment variables into the n8n container at startup.

**Tip:** For a strong password you can generate one with:
```bash
# Linux/macOS
openssl rand -base64 20

# Windows (PowerShell)
[System.Web.Security.Membership]::GeneratePassword(20,4)
```

---

### Step 3 — Connect kubectl to Your Kyma Cluster

1. Log in to the **SAP BTP Cockpit** (`cockpit.btp.cloud.sap`)
2. Navigate to your **Subaccount → Kyma Environment**
3. Click **"Download Kubeconfig"**
4. Save the file (e.g., `kyma-kubeconfig.yaml`) and point `kubectl` to it:

```bash
export KUBECONFIG=/path/to/kyma-kubeconfig.yaml

# Verify the connection
kubectl cluster-info
kubectl get nodes
```

You should see your Kyma cluster nodes listed with `Ready` status.

---

### Step 4 — Create the Namespace

All n8n resources live in the dedicated `n8n-v2` namespace.

```bash
kubectl apply -f namespace.yaml
```

Expected output:
```
namespace/n8n-v2 created
```

Confirm it exists:
```bash
kubectl get namespace n8n-v2
```

---

### Step 5 — Apply the Secret

The secret stores the n8n basic-auth credentials as Kubernetes-managed environment variables.

```bash
kubectl apply -f secret.yaml
```

Expected output:
```
secret/n8n-secret created
```

Verify (note: values are base64-encoded and not shown in plaintext by default):
```bash
kubectl get secret n8n-secret -n n8n-v2
```

---

### Step 6 — Deploy n8n

Apply the Deployment manifest to create the n8n pod:

```bash
kubectl apply -f deployment.yaml
```

Expected output:
```
deployment.apps/n8n created
```

Watch the pod come up (this may take 30–60 seconds as it pulls the image):
```bash
kubectl rollout status deployment/n8n -n n8n-v2
```

You should see:
```
deployment "n8n" successfully rolled out
```

Check the pod is `Running`:
```bash
kubectl get pods -n n8n-v2
```

```
NAME                   READY   STATUS    RESTARTS   AGE
n8n-xxxxxxxxxx-xxxxx   1/1     Running   0          45s
```

---

### Step 7 — Expose the Service

Create the internal ClusterIP service that routes traffic from the API gateway to the n8n pod:

```bash
kubectl apply -f service.yaml
```

Expected output:
```
service/n8n-service created
```

Verify:
```bash
kubectl get service n8n-service -n n8n-v2
```

```
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
n8n-service   ClusterIP   10.x.x.x        <none>        5678/TCP   10s
```

---

### Step 8 — Create the API Rule

The `APIRule` tells Kyma's API Gateway to forward external traffic for the host `n8n-v2` to the `n8n-service`:

```bash
kubectl apply -f apirule.yaml
```

Expected output:
```
apirule.gateway.kyma-project.io/n8n-api created
```

Check that the APIRule is in `Ready` state:
```bash
kubectl get apirule n8n-api -n n8n-v2
```

```
NAME      STATUS   HOST
n8n-api   Ready    n8n-v2.<your-kyma-cluster-domain>
```

> It may take 1–2 minutes for the gateway to reconcile and become `Ready`.

---

### Step 9 — Verify the Deployment

Run a full status check across all resources:

```bash
kubectl get all -n n8n-v2
```

You should see something similar to:

```
NAME                       READY   STATUS    RESTARTS   AGE
pod/n8n-xxxxxxxxxx-xxxxx   1/1     Running   0          2m

NAME                  TYPE        CLUSTER-IP    PORT(S)    AGE
service/n8n-service   ClusterIP   10.x.x.x      5678/TCP   2m

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/n8n   1/1     1            1           2m

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/n8n-xxxxxxxxxx   1         1         1       2m
```

Check the n8n application logs to confirm it started successfully:
```bash
kubectl logs -n n8n-v2 deployment/n8n --tail=50
```

Look for a line like:
```
Editor is now accessible via:
http://localhost:5678/
```

---

### Step 10 — Access n8n

Get your Kyma cluster's base domain:
```bash
kubectl get apirule n8n-api -n n8n-v2 -o jsonpath='{.spec.hosts[0]}'
```

The full URL will be:
```
https://n8n-v2.<your-kyma-cluster-id>.kyma.ondemand.com
```

Open the URL in your browser. You will be prompted for the basic-auth credentials you set in `secret.yaml`.

**Default credentials (change these!):**
- Username: `admin`
- Password: `ChangeMe123`

After logging in, you will be greeted by the n8n workflow editor and can start building automations.

---

## Configuration Reference

### Deployment Environment Variables (`deployment.yaml`)

| Variable | Value | Description |
|----------|-------|-------------|
| `N8N_BASIC_AUTH_ACTIVE` | `"true"` | Enables username/password protection |
| `N8N_HOST` | `"n8n-v2"` | Hostname n8n advertises for itself |
| `N8N_PROTOCOL` | `"https"` | Protocol used for generated URLs |
| `N8N_PORT` | `"5678"` | Port n8n listens on inside the container |
| `N8N_EDITOR_BASE_URL` | `"/"` | Base path for the web editor |
| `WEBHOOK_URL` | `"/"` | Base path for incoming webhooks |

### Secret Variables (`secret.yaml`)

| Variable | Description |
|----------|-------------|
| `N8N_BASIC_AUTH_USER` | Login username for n8n |
| `N8N_BASIC_AUTH_PASSWORD` | Login password for n8n |

### APIRule (`apirule.yaml`)

| Field | Value | Description |
|-------|-------|-------------|
| `gateway` | `kyma-system/kyma-gateway` | Uses the built-in Kyma Istio gateway |
| `hosts` | `["n8n-v2"]` | Subdomain on the cluster domain |
| `service.port` | `5678` | Forwards traffic to the ClusterIP service |
| `noAuth` | `true` | No OAuth/JWT at the gateway; relies on app-level auth |

---

## Updating n8n

To pull a newer version of the n8n image, update the image tag in `deployment.yaml` (or trigger a rollout if using `latest`):

```bash
# Force a new rollout (re-pulls the :latest image)
kubectl rollout restart deployment/n8n -n n8n-v2

# Watch progress
kubectl rollout status deployment/n8n -n n8n-v2
```

To pin to a specific version (recommended for production), edit `deployment.yaml`:
```yaml
image: n8nio/n8n:1.40.0   # replace with the desired version
```
Then re-apply:
```bash
kubectl apply -f deployment.yaml
```

---

## Teardown / Cleanup

To remove all n8n resources from the cluster, delete the namespace (this cascades to all resources inside it):

```bash
kubectl delete namespace n8n-v2
```

To remove resources individually:
```bash
kubectl delete -f apirule.yaml
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
kubectl delete -f secret.yaml
kubectl delete -f namespace.yaml
```

---

## Troubleshooting

### Pod is stuck in `Pending`

```bash
kubectl describe pod -n n8n-v2 -l app=n8n
```

Common causes:
- **Insufficient cluster resources** — the Kyma trial tier has limited CPU/memory; check node capacity with `kubectl top nodes`
- **Image pull failure** — verify internet connectivity from the cluster

### Pod is in `CrashLoopBackOff`

```bash
kubectl logs -n n8n-v2 deployment/n8n --previous
```

Common causes:
- Misconfigured environment variables
- Secret not found — ensure `secret.yaml` was applied before `deployment.yaml`

### APIRule is not `Ready`

```bash
kubectl describe apirule n8n-api -n n8n-v2
```

Common causes:
- Kyma API Gateway module not installed — check with `kubectl get module kyma-system 2>/dev/null || kubectl get apirules --all-namespaces`
- Wrong `gateway` reference — must match the installed gateway name

### Cannot reach the n8n URL

1. Confirm the APIRule is `Ready` (see above)
2. Check you're using `https://` (not `http://`)
3. Try curling from a local machine: `curl -I https://n8n-v2.<cluster-domain>`
4. Verify DNS has propagated: `nslookup n8n-v2.<cluster-domain>`

### Forgot the password

Update `secret.yaml` with a new password and re-apply, then restart the pod:

```bash
kubectl apply -f secret.yaml
kubectl rollout restart deployment/n8n -n n8n-v2
```

---

## Security Considerations

- **Change the default password** before any deployment — `ChangeMe123` is a placeholder only
- The `APIRule` uses `noAuth: true`, meaning Kyma's gateway does not enforce a token; authentication relies entirely on n8n's built-in basic auth
- Consider adding a Kyma `AuthorizationPolicy` or switching to JWT-based auth in `apirule.yaml` for stricter access control
- The deployment uses `n8nio/n8n:latest` — for production workloads, pin to a specific version to avoid unexpected breaking changes
- n8n workflows can execute arbitrary code (via the Code node and Execute Command node); ensure only trusted users have login access
- This setup has **no persistent volume** — workflow data is lost if the pod restarts. For production use, add a `PersistentVolumeClaim` or connect n8n to an external database (PostgreSQL/MySQL)

---

## License

This project is released under the [MIT License](LICENSE).
