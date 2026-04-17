# Canary Deployment Demo — Step by Step

## What is Canary?

Unlike Blue/Green (0% or 100%), Canary sends traffic **gradually** to the new version.

```
 STEP 1:  ████████████████████░░░░  80% old / 20% new   (test small)
 STEP 2:  ██████████░░░░░░░░░░░░░░  50% old / 50% new   (half-half)
 STEP 3:  ████░░░░░░░░░░░░░░░░░░░░  20% old / 80% new   (almost done)
 STEP 4:  ░░░░░░░░░░░░░░░░░░░░░░░░   0% old / 100% new  (fully live)

 █ = old version traffic
 ░ = new version traffic
```

---

## How It Works — Architecture

```
 ┌──────────────────────────────────────────────────────────── ┐
 │                       CLUSTER                               │
 │                                                             │
 │                   ┌─────────────┐                           │
 │                   │   INGRESS   │  ← Argo Rollouts controls │
 │                   │  (nginx)    │    the traffic % here     │
 │                   └──────┬──────┘                           │
 │                          │                                  │
 │              ┌───────────┴───────────┐                      │
 │              │                       │                      │
 │         80% traffic             20% traffic                │
 │              │                       │                      │
 │    ┌─────────▼────────┐   ┌─────────▼────────┐            │
 │    │ Stable Service   │   │ Canary Service   │            │
 │    │ (port 30090)     │   │ (port 30091)     │            │
 │    └─────────┬────────┘   └─────────┬────────┘            │
 │              │                       │                      │
 │    ┌─────────▼────────┐   ┌─────────▼────────┐            │
 │    │ OLD Pods (v1)    │   │ NEW Pods (v2)    │            │
 │    │ nginx:1.24       │   │ nginx:1.25       │            │
 │    │ BLUE page        │   │ ORANGE page      │            │
 │    │ 3 pods           │   │ 1 pod            │            │
 │    └──────────────────┘   └──────────────────┘            │
 │                                                             │
 │    After promote to 50%:                                    │
 │    OLD: 2 pods (50%)        NEW: 2 pods (50%)              │
 │                                                             │
 │    After promote to 80%:                                    │
 │    OLD: 1 pod (20%)         NEW: 3 pods (80%)              │
 │                                                             │
 │    After full promotion:                                    │
 │    OLD: 0 pods (gone)       NEW: 4 pods (100%)             │
 └─────────────────────────────────────────────────────────────┘
```

---

## Files in This Folder

| File | What It Does |
|---|---|
| `01-namespace.yaml` | Creates `canary-demo` namespace |
| `02-configmaps.yaml` | HTML pages for v1 (Blue), v2 (Orange), v3 (Purple) |
| `03-services.yaml` | Stable service (port 30090) + Canary service (port 30091) |
| `04-ingress-nginx-namespace.yaml` | Namespace for Nginx Ingress Controller |
| `05-ingress.yaml` | Ingress resource — Argo Rollouts controls traffic split here |
| `06-rollout.yaml` | Canary Rollout — deploys v1 (Blue) |
| `07-rollout-v2-update.yaml` | Updated Rollout — triggers canary with v2 (Orange) |
| `08-rollout-v3-update.yaml` | Updated Rollout — triggers canary with v3 (Purple) |
| `argocd/` | ArgoCD Application + Project files |

---

## Step-by-Step Commands (Run on KodeCloud Cluster)

---

### STEP 1: Install Nginx Ingress Controller

Canary needs an Ingress Controller to split traffic by percentage. This is NOT needed for Blue/Green.

```bash
# Create namespace
kubectl apply -f 04-ingress-nginx-namespace.yaml

# Install Nginx Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml

# Wait for it to be ready
kubectl get pods -n ingress-nginx -w
# Wait until STATUS = Running, then Ctrl+C
```

**Why?** Argo Rollouts tells the Nginx Ingress "send 20% traffic to canary pods, 80% to stable pods". Without Nginx Ingress, there's no way to split traffic by percentage.

---

### STEP 2: Install Argo Rollouts Controller

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Wait for it
kubectl get pods -n argo-rollouts -w
# Wait until Running, then Ctrl+C
```

---

### STEP 3: Install Argo Rollouts kubectl Plugin

```bash
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

# Verify
kubectl argo rollouts version
```

---

### STEP 4: Deploy v1 (Stable — Blue Page)

```bash
# Create namespace
kubectl apply -f 01-namespace.yaml

# Create all HTML pages (v1, v2, v3)
kubectl apply -f 02-configmaps.yaml

# Create services
kubectl apply -f 03-services.yaml

# Create Ingress
kubectl apply -f 05-ingress.yaml

# Deploy the Rollout (v1)
kubectl apply -f 06-rollout.yaml
```

Check status:

```bash
kubectl argo rollouts get rollout canary-app -n canary-demo --watch
# Wait for ✔ Healthy, then Ctrl+C
```

Check browser:

```
http://<NODE-IP>:30090  →  BLUE page v1.0 ✅ (stable)
http://<NODE-IP>:30091  →  BLUE page v1.0   (same, no canary yet)
```

---

### STEP 5: Deploy v2 (Trigger Canary — Orange Page)

```bash
kubectl apply -f 07-rollout-v2-update.yaml
```

Watch the canary progress:

```bash
kubectl argo rollouts get rollout canary-app -n canary-demo --watch
```

You will see:

```
Name:            canary-app
Status:          ॥ Paused
Strategy:        Canary
  Step:          1/6       ← At step 1 (setWeight: 20)
  SetWeight:     20        ← 20% traffic goes to canary
  ActualWeight:  20
Images:          nginx:1.24 (stable)     ← old version
                 nginx:1.25 (canary)     ← new version
Replicas:
  Desired:       4
  Current:       5
  Updated:       1         ← 1 canary pod
  Ready:         5
  Available:     5
```

**What's happening now:**

```
 ┌────────────────────────────────────────────┐
 │ 80% of requests  →  v1 BLUE (3 pods)      │
 │ 20% of requests  →  v2 ORANGE (1 pod)     │
 │                                            │
 │ Refresh browser multiple times:            │
 │   8 out of 10 → BLUE page                 │
 │   2 out of 10 → ORANGE page               │
 └────────────────────────────────────────────┘
```

Test it — refresh `http://<NODE-IP>:30090` many times. You'll see mostly BLUE, sometimes ORANGE.

---

### STEP 6: Promote to 50% (Step 2)

Happy with 20%? Move to 50%:

```bash
kubectl argo rollouts promote canary-app -n canary-demo
```

Check status:

```bash
kubectl argo rollouts get rollout canary-app -n canary-demo
```

```
  Step:          3/6       ← Now at step 3 (setWeight: 50)
  SetWeight:     50
  ActualWeight:  50
```

```
 ┌────────────────────────────────────────────┐
 │ 50% of requests  →  v1 BLUE (2 pods)      │
 │ 50% of requests  →  v2 ORANGE (2 pods)    │
 │                                            │
 │ Refresh browser:                           │
 │   5 out of 10 → BLUE                      │
 │   5 out of 10 → ORANGE                    │
 └────────────────────────────────────────────┘
```

---

### STEP 7: Promote to 80% (Step 3)

```bash
kubectl argo rollouts promote canary-app -n canary-demo
```

```
  SetWeight:     80
```

```
 ┌────────────────────────────────────────────┐
 │ 20% of requests  →  v1 BLUE (1 pod)       │
 │ 80% of requests  →  v2 ORANGE (3 pods)    │
 └────────────────────────────────────────────┘
```

After 30 seconds, it **auto-promotes to 100%** (configured in the YAML: `pause: {duration: 30s}`).

---

### STEP 8: Fully Promoted (100%)

```bash
kubectl argo rollouts get rollout canary-app -n canary-demo
```

```
Status:          ✔ Healthy
Images:          nginx:1.25 (stable)     ← v2 is now stable!
Replicas:
  Desired:       4
  Current:       4         ← only v2 pods, v1 is gone
```

```
 ┌────────────────────────────────────────────┐
 │ 100% of requests  →  v2 ORANGE (4 pods)   │
 │ v1 BLUE pods      →  GONE                 │
 └────────────────────────────────────────────┘
```

---

### TO ABORT AT ANY STEP (Rollback)

If something is wrong at 20% or 50%, abort:

```bash
kubectl argo rollouts abort canary-app -n canary-demo
```

```
 ┌────────────────────────────────────────────┐
 │ Canary pods   →  DELETED                   │
 │ 100% traffic  →  v1 BLUE (back to normal) │
 │ Impact: only 20% (or 50%) users saw v2    │
 └────────────────────────────────────────────┘
```

---

## Blue/Green vs Canary — Side by Side

```
 Blue/Green:
 ═══════════
 Before promote:  100% BLUE  │  0% GREEN (preview only, separate port)
 After promote:     0% BLUE  │ 100% GREEN
 ⚡ Instant switch. All or nothing.

 Canary:
 ════════
 Step 1:    80% STABLE │  20% CANARY  (same port! mixed traffic)
 Step 2:    50% STABLE │  50% CANARY
 Step 3:    20% STABLE │  80% CANARY
 Done:       0% STABLE │ 100% CANARY
 📊 Gradual. Users see both versions on SAME port.
```

---

## Quick Reference

| Action | Command |
|---|---|
| Check status | `kubectl argo rollouts get rollout canary-app -n canary-demo` |
| Watch live | `kubectl argo rollouts get rollout canary-app -n canary-demo --watch` |
| Promote to next step | `kubectl argo rollouts promote canary-app -n canary-demo` |
| Skip all steps (full promote) | `kubectl argo rollouts promote canary-app -n canary-demo --full` |
| Abort (rollback) | `kubectl argo rollouts abort canary-app -n canary-demo` |
| Undo | `kubectl argo rollouts undo canary-app -n canary-demo` |
| See pods | `kubectl get pods -n canary-demo` |
| Dashboard | `kubectl argo rollouts dashboard` → `http://localhost:3100` |

---

## Cleanup

```bash
kubectl delete namespace canary-demo
kubectl delete namespace argo-rollouts
kubectl delete namespace ingress-nginx
```
