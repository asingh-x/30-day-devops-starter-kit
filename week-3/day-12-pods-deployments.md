# Day 12 — Kubernetes Pods, Deployments, ReplicaSets, Rolling Updates, and Rollback

## What You Will Learn

- How to define and run Pods using YAML
- What labels and selectors do and why they matter
- What a ReplicaSet is and why you skip it in practice
- How to write a Deployment with rolling update strategy
- How to configure resource requests, limits, and health probes
- How to roll back a broken deployment

---

## 1. The Pod — Basic Unit of Kubernetes

A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share a network namespace and storage volumes. In practice, most Pods run a single container.

### Pod YAML Structure

```yaml
apiVersion: v1          # API group and version
kind: Pod               # Resource type
metadata:
  name: flask-pod                  # Unique name within the namespace
  namespace: default               # Namespace (omit to use default)
  labels:
    app: flask                     # Key-value pairs — used by selectors
    env: production
spec:                              # Desired state
  containers:
    - name: flask                  # Container name (must be unique within the Pod)
      image: python:3.11-slim      # Docker image
      ports:
        - containerPort: 5000      # Port the container listens on (informational only)
      command: ["python"]
      args: ["-m", "flask", "run", "--host=0.0.0.0"]
      env:
        - name: FLASK_APP
          value: "app.py"
      resources:
        requests:
          cpu: "100m"              # 100 millicores = 0.1 CPU
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

### Run a Pod from YAML vs kubectl run

```bash
# From YAML (preferred — repeatable, version-controllable)
kubectl apply -f flask-pod.yaml

# Imperative (quick testing only, not for production)
kubectl run flask-pod --image=python:3.11-slim --port=5000

# Inspect the running pod
kubectl get pods
kubectl describe pod flask-pod
kubectl logs flask-pod
kubectl exec -it flask-pod -- /bin/bash
```

---

## 2. Labels and Selectors

Labels are arbitrary key-value pairs attached to any Kubernetes object. Selectors filter objects by their labels. This is how Services find Pods, how Deployments manage ReplicaSets, and how ReplicaSets manage Pods.

```
Pod                           Service
┌─────────────────────┐       ┌───────────────────────┐
│ labels:             │       │ selector:             │
│   app: flask        │◄──────│   app: flask          │
│   env: production   │       │   env: production     │
└─────────────────────┘       └───────────────────────┘
```

Without a matching label, a Service will never route traffic to a Pod, and a Deployment will never manage a Pod. Label mismatches are one of the most common beginner mistakes.

```bash
# Filter pods by label
kubectl get pods -l app=flask
kubectl get pods -l app=flask,env=production

# Show labels in output
kubectl get pods --show-labels
```

---

## 3. ReplicaSet — What It Is, Why You Skip It

A ReplicaSet ensures that a specified number of Pod replicas are running at any time. If a Pod crashes, the ReplicaSet creates a replacement. If there are too many Pods, it deletes the extras.

```
ReplicaSet (desired: 3)
├── Pod-abc12  [Running]
├── Pod-def34  [Running]
└── Pod-xyz78  [Running]  ← if this crashes, RS creates a new one
```

**You almost never create a ReplicaSet directly.** A Deployment creates and manages ReplicaSets for you, and adds rolling update and rollback capabilities on top. Creating a bare ReplicaSet means you lose those features.

```bash
# You will see ReplicaSets created automatically by Deployments
kubectl get replicasets
kubectl describe replicaset <name>
```

---

## 4. Deployment — The Right Way to Run Pods

A Deployment manages ReplicaSets, which manage Pods. When you change the Deployment spec (e.g., update the image), it creates a new ReplicaSet and gradually moves traffic over.

```
Deployment
└── ReplicaSet (v2 — current)
    ├── Pod
    ├── Pod
    └── Pod
└── ReplicaSet (v1 — scaled to 0 after rollout, kept for rollback)
```

### Full Deployment YAML for a Python Flask App

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-deployment
  namespace: default
  labels:
    app: flask
spec:
  replicas: 3                          # Number of Pod replicas

  selector:
    matchLabels:
      app: flask                       # Must match template.metadata.labels

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                      # How many extra Pods above desired during update
      maxUnavailable: 1                # How many Pods can be down during update

  template:                            # Pod template — same structure as a Pod spec
    metadata:
      labels:
        app: flask                     # Must match spec.selector.matchLabels
        version: "1.0"
    spec:
      containers:
        - name: flask
          image: myrepo/flask-app:1.0.0
          ports:
            - containerPort: 5000
          env:
            - name: FLASK_ENV
              value: "production"
            - name: APP_PORT
              value: "5000"

          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"

          livenessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 10    # Wait 10s before first check
            periodSeconds: 15          # Check every 15s
            failureThreshold: 3        # Restart after 3 consecutive failures

          readinessProbe:
            httpGet:
              path: /ready
              port: 5000
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3        # Remove from Service endpoints after 3 failures

      restartPolicy: Always
```

```bash
kubectl apply -f flask-deployment.yaml
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

---

## 5. Rolling Update Strategy

When you update the image (or any Pod spec field), Kubernetes performs a rolling update:

1. Creates a new ReplicaSet with the new spec
2. Scales it up one Pod at a time (controlled by `maxSurge`)
3. Scales down the old ReplicaSet one Pod at a time (controlled by `maxUnavailable`)
4. Continues until all Pods run the new version

```
Before update:   [v1] [v1] [v1]
During update:   [v1] [v1] [v2]   (maxUnavailable=1, maxSurge=1)
                 [v1] [v2] [v2]
After update:    [v2] [v2] [v2]
```

### Trigger a Rolling Update

```bash
# Update image — this is the most common way to trigger a rolling update
kubectl set image deployment/flask-deployment flask=myrepo/flask-app:2.0.0

# Alternatively, edit the YAML and re-apply
kubectl edit deployment flask-deployment

# Or update the YAML file and re-apply (recommended for GitOps)
# Edit flask-deployment.yaml: change image tag to 2.0.0
kubectl apply -f flask-deployment.yaml
```

### Monitor the Rollout

```bash
# Watch rollout progress
kubectl rollout status deployment/flask-deployment

# View rollout history
kubectl rollout history deployment/flask-deployment

# View details of a specific revision
kubectl rollout history deployment/flask-deployment --revision=2

# Pause a rollout mid-way (to manually verify partial update)
kubectl rollout pause deployment/flask-deployment

# Resume the rollout
kubectl rollout resume deployment/flask-deployment
```

---

## 6. Rollback

If a new version is broken, roll back to the previous revision:

```bash
# Roll back to the previous version
kubectl rollout undo deployment/flask-deployment

# Roll back to a specific revision
kubectl rollout undo deployment/flask-deployment --to-revision=1

# Verify rollback completed
kubectl rollout status deployment/flask-deployment

# Check current image in use
kubectl get deployment flask-deployment -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Kubernetes keeps old ReplicaSets (scaled to 0) so that rollback is instant — it just scales the old ReplicaSet back up. The number of old ReplicaSets kept is controlled by `spec.revisionHistoryLimit` (default: 10).

```yaml
spec:
  revisionHistoryLimit: 5    # Keep only 5 old ReplicaSets
```

---

## 7. Resource Requests and Limits

```
requests  = what the container needs (used by the scheduler to pick a node)
limits    = the hard cap (container is killed/throttled if it exceeds this)
```

```yaml
resources:
  requests:
    cpu: "100m"       # 1000m = 1 full CPU core
    memory: "128Mi"   # Mibibytes — use Mi not MB
  limits:
    cpu: "500m"
    memory: "256Mi"
```

**Rules of thumb:**
- Set requests conservatively based on normal load
- Set limits 2-4x the request to handle bursts
- If a container exceeds its memory limit, it is OOMKilled (no warning)
- If a container exceeds its CPU limit, it is throttled (not killed)

```bash
# See resource usage
kubectl top pods
kubectl top nodes

# Describe a pod to see requests/limits
kubectl describe pod <pod-name>
```

---

## 8. Liveness and Readiness Probes

Kubernetes uses two probes to manage Pod lifecycle:

```
Liveness probe  → "Is the container alive? If not, restart it."
Readiness probe → "Is the container ready to receive traffic? If not, remove from Service endpoints."
```

### Three Probe Types

**httpGet** — makes an HTTP GET request; success = 200-399

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 10
  periodSeconds: 15
  failureThreshold: 3
  timeoutSeconds: 5
```

**exec** — runs a command inside the container; success = exit code 0

```yaml
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - "pgrep -x python"
  initialDelaySeconds: 5
  periodSeconds: 10
```

**tcpSocket** — checks if a TCP port accepts connections

```yaml
readinessProbe:
  tcpSocket:
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 10
```

### Minimal Flask App with Health Endpoints

```python
# app.py
from flask import Flask
app = Flask(__name__)

@app.route("/")
def index():
    import socket
    return f"Hello from {socket.gethostname()}"

@app.route("/health")
def health():
    return {"status": "ok"}, 200

@app.route("/ready")
def ready():
    # Add any readiness checks here (DB connection, etc.)
    return {"status": "ready"}, 200
```

---

## 9. Complete Reference Commands

```bash
# Create / update resources
kubectl apply -f flask-deployment.yaml

# Delete resources
kubectl delete -f flask-deployment.yaml
kubectl delete deployment flask-deployment

# Scale manually
kubectl scale deployment flask-deployment --replicas=5

# Get detailed state
kubectl describe deployment flask-deployment
kubectl describe pod <pod-name>

# Stream logs
kubectl logs -f <pod-name>
kubectl logs -f deployment/flask-deployment   # picks a random pod

# Get events for debugging
kubectl get events --sort-by='.lastTimestamp'

# Port forward for local testing
kubectl port-forward deployment/flask-deployment 8080:5000
```

---

## Exercises

### Exercise 1 — Write and Apply a Pod Manifest

Write a Pod YAML for `nginx:1.25`. Set the label `app=nginx`, expose port 80.
Apply it and confirm it is Running.

```bash
kubectl apply -f nginx-pod.yaml
kubectl get pod nginx-pod
kubectl describe pod nginx-pod
```

**Expected:** Pod status is `Running`, one container is ready.

---

### Exercise 2 — Create a Deployment and Scale It

Write a Deployment YAML for `nginx:1.25` with 2 replicas.
Apply it. Then scale it to 4 replicas using `kubectl scale`.
Then scale it back to 2 by editing the YAML and re-applying.

```bash
kubectl apply -f nginx-deployment.yaml
kubectl scale deployment nginx-deployment --replicas=4
kubectl get pods
# Edit YAML replicas: 2
kubectl apply -f nginx-deployment.yaml
kubectl get pods
```

**Expected:** Pod count changes from 2 to 4 and back to 2.

---

### Exercise 3 — Perform a Rolling Update and Watch It

Deploy `nginx:1.24` with 3 replicas.
In a separate terminal, run `kubectl get pods -w` to watch pod changes.
Update the image to `nginx:1.25`.
Observe the rolling replacement of old Pods with new ones.

```bash
# Terminal 1
kubectl get pods -w

# Terminal 2
kubectl set image deployment/nginx-deployment nginx=nginx:1.25
kubectl rollout status deployment/nginx-deployment
```

**Expected:** Old pods terminate one at a time as new ones become ready.

---

### Exercise 4 — Trigger and Recover from a Bad Rollout

Update the deployment image to `nginx:THIS_TAG_DOES_NOT_EXIST`.
Watch the rollout hang with ImagePullBackOff.
Roll back to the working version.

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:THIS_TAG_DOES_NOT_EXIST
kubectl rollout status deployment/nginx-deployment    # will hang
kubectl get pods                                       # ImagePullBackOff
kubectl rollout undo deployment/nginx-deployment
kubectl rollout status deployment/nginx-deployment    # should succeed
```

**Expected:** After undo, all pods return to the previous working image.

---

### Exercise 5 — Add Liveness and Readiness Probes

Add httpGet liveness and readiness probes to your nginx Deployment.
- Liveness: GET `/` on port 80, initialDelay 5s, period 10s
- Readiness: GET `/` on port 80, initialDelay 3s, period 5s

Apply and verify the probes appear in the pod description.

```bash
kubectl apply -f nginx-deployment.yaml
kubectl describe pod <pod-name> | grep -A 10 "Liveness\|Readiness"
```

**Expected:** Both probes appear in the pod description with correct config.

---

### Exercise 6 — Inspect Rollout History and Revision Details

Make 3 different image updates to your deployment (use nginx:1.23, 1.24, 1.25 in sequence).
Add `--record` annotation to your apply commands (or use `kubectl annotate`).
View rollout history and details of each revision.
Roll back to revision 2.

```bash
# Note: --record was deprecated in Kubernetes 1.19 and removed in 1.28+.
# To record a change cause, use: kubectl annotate deployment <name> kubernetes.io/change-cause="your message"

kubectl set image deployment/nginx-deployment nginx=nginx:1.23
kubectl set image deployment/nginx-deployment nginx=nginx:1.24
kubectl set image deployment/nginx-deployment nginx=nginx:1.25

kubectl rollout history deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment --revision=2
kubectl rollout undo deployment/nginx-deployment --to-revision=2

kubectl get deployment nginx-deployment -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Expected:** Current image is `nginx:1.24` (revision 2).

---

## Key Takeaways

- A Pod is the smallest unit; a Deployment wraps it with lifecycle management
- Labels connect Deployments to ReplicaSets to Pods to Services — mismatched labels break everything
- Never create a ReplicaSet directly; always use a Deployment
- `maxSurge` and `maxUnavailable` control rolling update speed and availability
- `kubectl rollout undo` is fast because it just scales an old ReplicaSet back up
- Requests are for scheduling; limits are hard caps — memory over limit = OOMKill
- Liveness probes restart unhealthy containers; readiness probes remove them from Service traffic
- Always have `/health` and `/ready` endpoints in any app you deploy to Kubernetes
