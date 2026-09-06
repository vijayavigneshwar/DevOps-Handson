# Kubernetes Study Notes — Topic 4: Pods

> Personal hands-on reference. Environment: Windows + Docker Desktop (Kubernetes enabled), kubectl via PowerShell.

---

## 1. Write and run a basic Pod manifest

**Goal:** Create the simplest possible workload — a single-container Pod.

**pod.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
```

**Commands**
```
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod nginx-pod
kubectl logs nginx-pod
```

**What to check:** `STATUS: Running`, `READY: 1/1`.

**Note (Windows/PowerShell):** `jsonpath` queries need **double quotes**, not single quotes:
```
kubectl get pod nginx-pod -o jsonpath="{.status.phase}"
```
Single quotes get mangled by PowerShell before kubectl even sees them.

---

## 2. Pod lifecycle phases & container states

**Two different things:**
- **Pod phase** — overall status: `Pending`, `Running`, `Succeeded`, `Failed`, `Unknown`
- **Container state** — per-container detail: `waiting`, `running`, `terminated` (each with a reason)

**Commands**
```
kubectl get pods                     # STATUS column = phase
kubectl describe pod nginx-pod       # Containers section = state, last state, restart count
```

**Key experiment — watch a phase transition live:**
```
kubectl delete pod nginx-pod
kubectl get pods                     # pod is gone — bare Pods are NOT self-healing
kubectl apply -f pod.yaml
kubectl get pods -w                  # watch: (nothing) -> ContainerCreating -> Running
```
Ctrl+C to stop watching once `Running`.

**Takeaway:** A bare Pod has no controller behind it — if it dies, nothing recreates it. This is exactly why Deployments/ReplicaSets exist (Topic 5).

---

## 3. Multi-container Pods (sidecar pattern)

**Goal:** Run two containers in one Pod that share network + storage.

**multi-pod.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  containers:
  - name: main-app
    image: busybox
    command: ["sh", "-c", "while true; do echo $(date) 'App is running' >> /var/log/app.log; sleep 5; done"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  - name: log-sidecar
    image: busybox
    command: ["sh", "-c", "tail -f /var/log/app.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  volumes:
  - name: shared-logs
    emptyDir: {}
```

**Commands**
```
kubectl apply -f multi-pod.yaml
kubectl get pods                                  # look for READY 2/2
kubectl logs sidecar-demo -c log-sidecar          # -c required with multiple containers
kubectl logs sidecar-demo -c main-app
kubectl exec -it sidecar-demo -c main-app -- sh   # exec into a specific container
```

**Takeaway:** Containers in the same Pod share network namespace and any mounted volumes, but have separate processes/logs. This is the sidecar pattern — one container does the main job, another supports it (logging, proxying, syncing, etc.).

---

## 4. Resource requests & limits (CPU/memory)

**Goal:** See Kubernetes enforce memory limits via OOM kill.

**resource-pod.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: stress-app
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
        cpu: "100m"
      limits:
        memory: "100Mi"
        cpu: "200m"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```
(Deliberately asks for 150Mi against a 100Mi limit.)

**Commands**
```
kubectl apply -f resource-pod.yaml
kubectl get pods -w
kubectl describe pod resource-demo
```

**What happened in practice:**
```
resource-demo   1/1   Running             0
resource-demo   0/1   OOMKilled           0
resource-demo   1/1   Running             1 (2s ago)
resource-demo   0/1   OOMKilled           1
...
resource-demo   0/1   CrashLoopBackOff    3
```

**Takeaway:**
- `requests` = what's reserved/guaranteed for scheduling.
- `limits` = hard ceiling; exceeding the **memory** limit gets the container `OOMKilled` (CPU limit just throttles, doesn't kill).
- Repeated failures escalate into `CrashLoopBackOff` — Kubernetes' way of spacing out retries with exponential backoff.
- `OOMKilled` in real life = app needs more memory or has a leak.

**Note:** `kubectl top pod` won't work here — Docker Desktop's Kubernetes doesn't ship Metrics Server by default. That gets installed later in the Autoscaling topic.

**Cleanup:** `kubectl delete pod resource-demo`

---

## 5. Liveness, readiness, and startup probes

**Goal:** Watch Kubernetes detect an unhealthy container and self-heal.

**probes-pod.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probes-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

**Commands**
```
kubectl apply -f probes-pod.yaml
kubectl get pods -w                    # wait for READY 1/1

# Break it deliberately
kubectl exec -it probes-demo -- sh -c "chmod 000 /usr/share/nginx/html/index.html"

kubectl get pods -w                    # watch RESTARTS increase
kubectl describe pod probes-demo       # check Events
```

**Actual Events observed:**
```
Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 403
Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 403
Normal   Killing    Container nginx failed liveness probe, will be restarted
```

**Takeaway:**
- **Readiness** = "is it ready for traffic right now?" — fails fast (checked every 5s here), pod pulled from Service traffic rotation, but the container itself isn't touched.
- **Liveness** = "is it alive at all?" — if it keeps failing, kubelet kills and restarts the container.
- They are independent checks with independent consequences. A container restart resets it back to the image's original state (which is why it fixed itself after restart).

**Cleanup:** `kubectl delete pod probes-demo`

---

## 6. Init containers

**Goal:** Run setup logic that must finish before the main container starts.

**init-pod.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: wait-for-setup
    image: busybox
    command: ["sh", "-c", "echo 'Running setup...'; sleep 10; echo 'Setup complete' > /work-dir/ready.txt"]
    volumeMounts:
    - name: workdir
      mountPath: /work-dir
  containers:
  - name: main-app
    image: busybox
    command: ["sh", "-c", "cat /work-dir/ready.txt; sleep 3600"]
    volumeMounts:
    - name: workdir
      mountPath: /work-dir
  volumes:
  - name: workdir
    emptyDir: {}
```

**Commands**
```
kubectl apply -f init-pod.yaml
kubectl get pods -w                       # watch Init:0/1 for ~10s, then Running
kubectl logs init-demo -c wait-for-setup
kubectl logs init-demo -c main-app        # should print "Setup complete"
```

**Takeaway:** Init containers run sequentially and must exit successfully (`exit 0`) before any main container starts. Common uses: wait for a dependency, prep config, run migrations.

**Cleanup:** `kubectl delete pod init-demo`

---

## 7. Restart policies

**Goal:** See the difference between `Always`, `OnFailure`, and `Never`.

```
kubectl get pod nginx-pod -o jsonpath="{.spec.restartPolicy}"   # default = Always
```

**restart-policy-demo.yaml (Never)**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-never-demo
spec:
  restartPolicy: Never
  containers:
  - name: quick-exit
    image: busybox
    command: ["sh", "-c", "echo 'Running once and exiting'; exit 1"]
```

**Commands**
```
kubectl apply -f restart-policy-demo.yaml
kubectl get pods -w      # sits at "Error" — no restart, no CrashLoopBackOff
```

Change `restartPolicy: Never` → `OnFailure`, rename the pod, reapply:
```
kubectl delete pod restart-never-demo
kubectl apply -f restart-policy-demo.yaml
kubectl get pods -w      # this time it DOES restart -> CrashLoopBackOff
```

**Takeaway:**
| Policy | Behavior | Typical use |
|---|---|---|
| `Always` (default) | Always restarts on exit | Deployments, long-running services |
| `OnFailure` | Restarts only on non-zero exit | Jobs that should retry on failure |
| `Never` | Never restarts | One-off runs, debugging |

**Important nuance:** `CrashLoopBackOff` is **not an error type itself** — it's a *label for the retry-delay state* that appears whenever a container keeps failing and getting restarted repeatedly (under `Always` or `OnFailure`). The real root cause is always something else — check `describe`/`logs` to find it.

**Cleanup:** `kubectl delete pod restart-never-demo restart-onfailure-demo`

---

## 8. Debugging a crashing pod (logs, describe, exec)

**Goal:** Practice the standard debugging checklist end-to-end.

**debug-demo.yaml (deliberately broken)**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-demo
spec:
  containers:
  - name: broken-app
    image: busybox
    command: ["nonexistent-command"]
```

**Debugging checklist:**
```
kubectl apply -f debug-demo.yaml
kubectl get pods                      # STATUS: CrashLoopBackOff / StartError
kubectl describe pod debug-demo       # check Events for the real error
kubectl logs debug-demo               # may be empty -> runtime-level failure, not app-level
```

**Fix and verify:**
```yaml
command: ["sh", "-c", "echo hello; sleep 3600"]
```
```
kubectl delete pod debug-demo
kubectl apply -f debug-demo.yaml
kubectl get pods                      # now Running
```

**Standard debugging order to remember:**
1. `kubectl get pods` — what's the status?
2. `kubectl describe pod <name>` — check **Events** for the real reason
3. `kubectl logs <name>` (add `-c <container>` if multi-container) — check app-level output
4. `kubectl exec -it <name> -- sh` — poke around live if needed

**Final cleanup for the whole topic:**
```
kubectl delete pod debug-demo nginx-pod sidecar-demo
kubectl get pods         # should return "No resources found"
```

---

## Topic Summary

| # | Task | Key concept |
|---|---|---|
| 1 | Basic Pod manifest | apiVersion/kind/metadata/spec structure |
| 2 | Lifecycle phases & states | Pod phase vs container state; bare Pods aren't self-healing |
| 3 | Multi-container Pods | Shared network/volumes, separate processes (sidecar pattern) |
| 4 | Resource requests/limits | OOMKilled, CrashLoopBackOff |
| 5 | Probes | Liveness vs readiness — different checks, different consequences |
| 6 | Init containers | Sequential setup before main container starts |
| 7 | Restart policies | Always / OnFailure / Never |
| 8 | Debugging | get → describe → logs → exec, in that order |
