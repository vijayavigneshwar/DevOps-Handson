# Kubernetes Study Notes — Topic 5: Deployments & ReplicaSets

> Personal hands-on reference. Environment: Windows + Docker Desktop (Kubernetes enabled), kubectl via PowerShell.

---

## 1. ReplicaSets — maintaining desired pod count

**Goal:** See a ReplicaSet's core job: keep exactly N pods matching a label, forever.

**replicaset.yaml**
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-rs
  template:
    metadata:
      labels:
        app: nginx-rs
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

**Commands**
```
kubectl apply -f replicaset.yaml
kubectl get pods -l app=nginx-rs
kubectl get replicaset

# key experiment
kubectl delete pod <one-of-the-three-pod-names>
kubectl get pods -l app=nginx-rs -w     # a new pod appears automatically to restore count = 3

kubectl describe replicaset nginx-rs    # Events shows SuccessfulCreate
```

**Takeaway:** A ReplicaSet continuously reconciles: *"I want N pods matching this label — if there aren't N, create more."* That's its entire job — no versioning, no rollout logic.

**Cleanup:** `kubectl delete replicaset nginx-rs`

---

## 2. Deployments — managing stateless apps

**Goal:** See the ownership chain: Deployment → ReplicaSet → Pods.

**deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-deploy
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

**Commands**
```
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get replicasets                 # notice the hash suffix naming pattern
kubectl get pods -l app=nginx-deploy

kubectl describe deployment nginx-deployment   # look for "NewReplicaSet"

# self-healing works one layer up too
kubectl delete pod <one-pod-name>
kubectl get pods -l app=nginx-deploy    # bounces back to 3

# scaling
kubectl scale deployment nginx-deployment --replicas=5
```

**Why do we need ReplicaSets if Deployments do the work?**
A ReplicaSet's only job is maintaining pod count. A Deployment sits on top and adds **version management** — rolling updates and rollback — which a ReplicaSet alone can't do.

When you update a Deployment's image, Kubernetes does **not** edit the existing ReplicaSet. Instead:
1. Creates a **brand-new ReplicaSet** for the new version
2. Scales the **new** RS up gradually
3. Scales the **old** RS down gradually
4. Keeps the old RS around at 0 replicas — that's your rollback history

So: **ReplicaSet = low-level engine that keeps pod counts correct. Deployment = higher-level controller that manages which ReplicaSet is active and transitions between them.**

---

## 3. Rolling updates (maxSurge / maxUnavailable)

**Goal:** Watch the RollicaSet handoff happen live during an update.

```
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

kubectl get replicasets -w
# observe: new RS appears and scales UP, old RS scales DOWN, simultaneously

kubectl get pods -l app=nginx-deploy -o jsonpath="{.items[*].spec.containers[*].image}"
# all pods now show nginx:1.26

kubectl rollout status deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment
```

**Takeaway:** `RollingUpdate` (the default strategy) replaces old pods with new ones incrementally — old and new versions run **simultaneously** for a short window, giving zero downtime. Controlled by `maxSurge` (how many extra pods above desired count during rollout) and `maxUnavailable` (how many can be down at once).

---

## 4. Rolling back a Deployment

**Goal:** Use rollout history as a safety net for bad releases.

```
kubectl rollout history deployment/nginx-deployment

# deliberately deploy a broken image tag
kubectl set image deployment/nginx-deployment nginx=nginx:1.99-does-not-exist

kubectl get pods -l app=nginx-deploy
# some pods stuck in ImagePullBackOff / ErrImagePull

kubectl rollout status deployment/nginx-deployment
# hangs/times out — old (working) RS is NOT scaled down until new pods are healthy

kubectl rollout undo deployment/nginx-deployment

kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx-deploy -o jsonpath="{.items[*].spec.containers[*].image}"
# back to nginx:1.26, all Running

kubectl rollout history deployment/nginx-deployment
# rollback recorded as a new revision
```

**Takeaway:** Kubernetes won't scale down a healthy old ReplicaSet until the new one proves healthy — a bad rollout can't take down the whole app on its own. `kubectl rollout undo` reverts to the last known-good ReplicaSet in one command.

---

## 5. Manual scaling & ReplicaSet ownership

**Goal:** Prove that scale should always go through the Deployment, not the ReplicaSet.

```
kubectl get deployment nginx-deployment
kubectl get replicasets

kubectl scale deployment nginx-deployment --replicas=2
kubectl get pods -l app=nginx-deploy
kubectl get replicasets           # active RS count matches

kubectl scale deployment nginx-deployment --replicas=6
kubectl get pods -l app=nginx-deploy

# try scaling the RS directly instead
kubectl get replicasets
kubectl scale replicaset <active-rs-name> --replicas=1
kubectl get pods -l app=nginx-deploy -w
# within seconds, pod count snaps back up to 6

kubectl describe deployment nginx-deployment
```

**Takeaway:** The Deployment controller continuously reconciles the ReplicaSet back to *its own* desired state — any direct edit to the RS is treated as drift and gets corrected. **Always scale through the Deployment, never the ReplicaSet directly.** Same reconciliation pattern as Pods → ReplicaSets, one layer up.

---

## 6. Deployment strategies: RollingUpdate vs Recreate

**Goal:** Compare the two update strategies side by side.

**deployment.yaml (with Recreate strategy)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nginx-deploy
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      containers:
      - name: nginx
        image: nginx:1.26
        ports:
        - containerPort: 80
```

**Commands**
```
kubectl apply -f deployment.yaml
kubectl set image deployment/nginx-deployment nginx=nginx:1.27

kubectl get pods -l app=nginx-deploy -w
# ALL old pods terminate first (pod count briefly hits 0),
# THEN all new pods get created together

kubectl get pods -l app=nginx-deploy -o jsonpath="{.items[*].spec.containers[*].image}"
```

**Takeaway:**
| Strategy | Behavior | Tradeoff |
|---|---|---|
| `RollingUpdate` (default) | Old and new pods overlap, gradual handoff | Zero downtime, but two versions run simultaneously for a bit |
| `Recreate` | All old pods killed, then all new pods created | Brief downtime, but guarantees only one version ever runs at once |

Use `Recreate` when versions can't safely coexist (e.g. incompatible schema/singleton workloads).

---

## 7. Pause / resume a rollout

**Goal:** Batch multiple spec changes into a single rollout instead of triggering one per change.

```
kubectl apply -f deployment.yaml
kubectl get pods -l app=nginx-deploy      # wait for all Running

kubectl rollout pause deployment/nginx-deployment

# make multiple changes while paused - nothing rolls out yet
kubectl set image deployment/nginx-deployment nginx=nginx:1.27
kubectl set resources deployment/nginx-deployment -c=nginx --limits=cpu=200m,memory=256Mi

kubectl get pods -l app=nginx-deploy      # unchanged - still old pods
kubectl get replicasets                   # no new RS created yet

kubectl rollout resume deployment/nginx-deployment
kubectl get pods -l app=nginx-deploy -w
# single new ReplicaSet created with BOTH changes applied together

kubectl describe deployment nginx-deployment
# confirm new image + new resource limits both present
```

**Takeaway:** Pausing suspends the reconciliation loop from acting on spec changes; resuming flushes everything queued into one rollout. Useful for batching several config changes (image + resources + env vars, etc.) into a single, cleaner rollout instead of multiple disruptive ones.

**Final cleanup:**
```
kubectl delete deployment nginx-deployment
kubectl get pods
```

---

## Topic Summary

| # | Task | Key concept |
|---|---|---|
| 1 | ReplicaSets | Maintains exact pod count via reconciliation — no versioning |
| 2 | Deployments | Manages ReplicaSets; ownership chain Deployment → RS → Pods |
| 3 | Rolling updates | New RS scales up, old RS scales down simultaneously (zero downtime) |
| 4 | Rollback | `rollout undo` reverts to last healthy RS; bad rollouts are auto-protected |
| 5 | Manual scaling | Always scale the Deployment, not the RS — direct RS edits get overridden |
| 6 | Strategies | RollingUpdate (no downtime, overlap) vs Recreate (downtime, no overlap) |
| 7 | Pause/resume | Batch multiple changes into a single rollout |

### Deployment vs ReplicaSet — the one-line answer
> ReplicaSet = keeps N pods running. Deployment = manages *which* ReplicaSet is active and handles the transition between versions (rolling updates, rollback). You'll almost always work with Deployments directly and let them manage ReplicaSets for you.
