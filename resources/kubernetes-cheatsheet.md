# Kubernetes Cheatsheet

A quick-reference guide for kubectl and Kubernetes concepts used in DevOps workflows.

---

## kubectl Basics

```bash
kubectl get <resource>            # List resources
kubectl describe <resource> <name> # Detailed info and events
kubectl apply -f file.yaml        # Create or update from file
kubectl delete -f file.yaml       # Delete from file
kubectl delete <resource> <name>  # Delete by name
kubectl logs <pod>                # View pod logs
kubectl exec -it <pod> -- bash    # Shell into a pod
kubectl explain <resource>        # Show field docs (e.g. kubectl explain pod.spec)
kubectl api-resources             # List all available resource types
```

---

## Short Name Aliases

| Full Name             | Short Name |
|-----------------------|------------|
| pods                  | po         |
| deployments           | deploy     |
| services              | svc        |
| namespaces            | ns         |
| configmaps            | cm         |
| secrets               | secret     |
| ingresses             | ing        |
| serviceaccounts       | sa         |
| persistentvolumeclaims| pvc        |
| nodes                 | no         |
| replicasets           | rs         |

```bash
# Short name examples
kubectl get po
kubectl get deploy -n production
kubectl describe svc myservice
kubectl get cm --all-namespaces
```

---

## Namespace Commands

```bash
kubectl get ns                                      # List all namespaces
kubectl create namespace staging                    # Create namespace
kubectl delete namespace staging                    # Delete namespace and all resources
kubectl get all -n staging                          # All resources in a namespace
kubectl get pods --all-namespaces                   # Pods across all namespaces
kubectl get pods -A                                 # Short form of --all-namespaces

# Set default namespace for current context
kubectl config set-context --current --namespace=staging
```

---

## Pod Commands

```bash
kubectl get pods                                    # List pods in current namespace
kubectl get pods -o wide                            # Include node and IP info
kubectl get pods -w                                 # Watch for changes
kubectl get pods -l app=nginx                       # Filter by label
kubectl get pods --field-selector=status.phase=Running  # Filter by status

kubectl describe pod mypod                          # Detailed pod info + events
kubectl logs mypod                                  # Logs from pod
kubectl logs mypod -c mycontainer                   # Logs from specific container
kubectl logs -f mypod                               # Follow logs
kubectl logs --previous mypod                       # Logs from previous (crashed) instance
kubectl logs --tail=100 mypod                       # Last 100 lines

kubectl exec -it mypod -- bash                      # Shell into pod
kubectl exec -it mypod -c mycontainer -- bash       # Shell into specific container
kubectl exec mypod -- env                           # Run command, print env vars
kubectl exec mypod -- cat /etc/config/app.conf      # Read a file

kubectl delete pod mypod                            # Delete pod (recreated by deployment)
kubectl delete pod mypod --grace-period=0 --force   # Force delete

# Run a temporary debug pod
kubectl run debug --image=busybox --rm -it --restart=Never -- sh
kubectl run curl --image=curlimages/curl --rm -it --restart=Never -- sh
```

---

## Deployment Commands

```bash
# Create and apply
kubectl apply -f deployment.yaml
kubectl create deployment nginx --image=nginx:latest     # Imperative create

# View deployments
kubectl get deploy
kubectl describe deploy nginx
kubectl get deploy nginx -o yaml                         # See full YAML

# Scale
kubectl scale deployment nginx --replicas=5
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80

# Update image (triggers rolling update)
kubectl set image deployment/nginx nginx=nginx:1.25

# Rollout management
kubectl rollout status deployment/nginx                  # Check rollout progress
kubectl rollout history deployment/nginx                 # View revision history
kubectl rollout history deployment/nginx --revision=2    # Details of revision 2
kubectl rollout undo deployment/nginx                    # Roll back to previous
kubectl rollout undo deployment/nginx --to-revision=2   # Roll back to specific revision
kubectl rollout pause deployment/nginx                   # Pause a rollout
kubectl rollout resume deployment/nginx                  # Resume a paused rollout

# Restart all pods in a deployment (rolling restart, no downtime)
kubectl rollout restart deployment/nginx
```

---

## Service Commands

```bash
kubectl get svc                                     # List services
kubectl describe svc myservice                      # Service details + endpoints
kubectl get endpoints myservice                     # Check which pods are targeted

# Expose a deployment as a service
kubectl expose deployment nginx --port=80 --target-port=8080 --type=ClusterIP
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Port-forward to test locally (without a LoadBalancer)
kubectl port-forward svc/myservice 8080:80          # Access service on localhost:8080
kubectl port-forward pod/mypod 8080:8080            # Port-forward to a pod

# Delete service
kubectl delete svc myservice
```

---

## ConfigMap and Secret Commands

```bash
# ConfigMap
kubectl get cm
kubectl describe cm myconfig
kubectl create configmap myconfig --from-literal=DB_HOST=localhost
kubectl create configmap myconfig --from-file=config.properties
kubectl create configmap myconfig --from-env-file=.env
kubectl delete cm myconfig

# Secret
kubectl get secrets
kubectl describe secret mysecret
kubectl create secret generic mysecret --from-literal=password=s3cr3t
kubectl create secret generic mysecret --from-file=./tls.crt --from-file=./tls.key
kubectl create secret tls mytls --cert=tls.crt --key=tls.key    # TLS secret

# Decode a secret value
kubectl get secret mysecret -o jsonpath='{.data.password}' | base64 --decode
```

---

## Ingress Commands

```bash
kubectl get ing                                      # List ingresses
kubectl describe ing myingress                       # Ingress details
kubectl apply -f ingress.yaml
kubectl delete ing myingress

# Check ingress controller (common: nginx ingress)
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

---

## Context and Config Commands

```bash
# View config
kubectl config view                                  # Full kubeconfig
kubectl config get-contexts                          # List all contexts
kubectl config current-context                       # Show current context

# Switch context
kubectl config use-context my-cluster
kubectl config use-context arn:aws:eks:us-east-1:123456:cluster/mycluster  # EKS

# Set namespace for context
kubectl config set-context --current --namespace=production

# Merge kubeconfig files
KUBECONFIG=~/.kube/config:~/new-cluster-config kubectl config view --merge --flatten > ~/.kube/config
```

---

## Common YAML Snippets

### Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: default
  labels:
    app: myapp
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      ports:
        - containerPort: 8080
      env:
        - name: ENV
          value: production
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          ports:
            - containerPort: 8080
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
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: default
spec:
  selector:
    app: myapp              # Must match pod labels
  ports:
    - protocol: TCP
      port: 80              # Port the service listens on
      targetPort: 8080      # Port the pod listens on
  type: ClusterIP           # ClusterIP | NodePort | LoadBalancer
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: default
data:
  DB_HOST: "postgres.default.svc.cluster.local"
  DB_PORT: "5432"
  LOG_LEVEL: "info"
  app.properties: |
    server.port=8080
    feature.flag=true
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
```

---

## Debugging Flow: Pod Not Starting

```
1. Check pod status
   kubectl get pods

   Common statuses:
   - Pending        → not scheduled yet (resource constraints, node issues)
   - CrashLoopBackOff → container starts then crashes repeatedly
   - ImagePullBackOff → cannot pull the image (bad tag, no credentials)
   - OOMKilled      → out of memory, increase limits
   - Error          → container exited with non-zero code

2. Describe the pod to see events
   kubectl describe pod <pod-name>

   Look at the "Events" section at the bottom:
   - "Failed to pull image" → check image name and registry credentials
   - "Insufficient cpu/memory" → node doesn't have enough resources
   - "Back-off restarting failed container" → app is crashing on startup

3. Check logs
   kubectl logs <pod-name>
   kubectl logs <pod-name> --previous      # If pod has already crashed

4. Check cluster events
   kubectl get events --sort-by='.lastTimestamp' -n default

5. Check if the container can start interactively
   # Override entrypoint to drop into shell
   kubectl run debug --image=myapp:1.0 -it --rm --restart=Never -- sh

6. Check resource quotas
   kubectl describe namespace default      # Check if namespace has resource quota
   kubectl get resourcequota -n default

7. Check node health
   kubectl get nodes
   kubectl describe node <node-name>       # Look for conditions and events
```

---

## Resource Units Reference

### CPU

| Unit   | Meaning           | Notes                                 |
|--------|-------------------|---------------------------------------|
| `1`    | 1 full CPU core   | Equivalent to 1 vCPU on cloud VMs     |
| `0.5`  | Half a CPU core   | Same as 500m                          |
| `100m` | 100 millicores    | 0.1 CPU — typical for small apps      |
| `500m` | 500 millicores    | 0.5 CPU                               |

### Memory

| Unit  | Meaning              | Notes                          |
|-------|----------------------|--------------------------------|
| `Mi`  | Mebibytes (1024^2)   | Use this in Kubernetes         |
| `Gi`  | Gibibytes (1024^3)   | Use this in Kubernetes         |
| `MB`  | Megabytes (1000^2)   | Not the same as Mi             |
| `GB`  | Gigabytes (1000^3)   | Not the same as Gi             |

```
128Mi  ≈ 134 MB
256Mi  ≈ 268 MB
1Gi    ≈ 1.07 GB
```

### Common Resource Sizing Patterns

```yaml
# Small (sidecars, batch jobs)
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"

# Medium (web apps, APIs)
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# Large (data processing, ML inference)
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
```

> Rule of thumb: always set `requests` (used for scheduling) and `limits` (prevents runaway resource use).
> Set limits too low and the container gets OOMKilled or CPU-throttled.
