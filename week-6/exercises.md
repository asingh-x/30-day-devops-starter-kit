# Week 6 Capstone — Production-Ready Kubernetes

This capstone ties together everything from the full 6-week program. By the end, the Flask application from Week 4 is running securely in Kubernetes with proper RBAC, packaged with Helm, scanned in CI/CD, and deployed via GitOps. Work through the parts in order — each one builds on the previous.

---

## Part 1: Secure the Kubernetes Cluster (Day 26)

**Goal**: harden the cluster so that the Flask application runs with the minimum permissions required and cannot affect other namespaces.

### 1.1 Create the namespace

```bash
kubectl create namespace flask-prod
```

### 1.2 Create a least-privilege ServiceAccount and Role

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flask-deployer
  namespace: flask-prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: flask-deployer-role
  namespace: flask-prod
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "deployments"]
    verbs: ["get", "list", "watch", "create", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: flask-deployer-binding
  namespace: flask-prod
subjects:
  - kind: ServiceAccount
    name: flask-deployer
    namespace: flask-prod
roleRef:
  kind: Role
  name: flask-deployer-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f serviceaccount.yaml
```

Verify that the ServiceAccount cannot delete pods (it must output `no`):

```bash
kubectl auth can-i delete pods \
  --as=system:serviceaccount:flask-prod:flask-deployer \
  -n flask-prod
```

### 1.3 Move secrets out of YAML files

Never commit credentials to Git. Create the secret via the CLI:

```bash
kubectl create secret generic flask-db-credentials \
  --from-literal=DB_USER=flaskuser \
  --from-literal=DB_PASSWORD=changeme \
  -n flask-prod
```

Reference it in the Deployment using `envFrom` or individual `env` entries with `secretKeyRef`.

### 1.4 Add a SecurityContext to the Deployment

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - name: flaskapp
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
```

### 1.5 Add NetworkPolicy

Apply a default-deny policy first, then an explicit allow policy:

```yaml
# default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: flask-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# allow-flaskapp.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-flaskapp-ingress
  namespace: flask-prod
spec:
  podSelector:
    matchLabels:
      app: flaskapp
  policyTypes:
    - Ingress
  ingress:
    - ports:
        - port: 8080
```

```bash
kubectl apply -f default-deny.yaml
kubectl apply -f allow-flaskapp.yaml
```

---

## Part 2: Package with Helm (Day 27)

**Goal**: replace raw YAML manifests with a Helm chart that supports multiple environments through values files.

### 2.1 Create the chart

```bash
helm create flaskapp
```

Clean out the default templates and replace them with a Deployment, Service, and ConfigMap for the Flask application.

### 2.2 values.yaml (defaults)

```yaml
replicaCount: 1

image:
  repository: <your-ecr-url>/flaskapp
  tag: latest
  pullPolicy: IfNotPresent

service:
  port: 8080

resources: {}
```

### 2.3 values-prod.yaml (production overrides)

```yaml
replicaCount: 3

resources:
  limits:
    memory: "128Mi"
    cpu: "100m"
  requests:
    memory: "64Mi"
    cpu: "50m"
```

### 2.4 Install

```bash
helm install flaskapp ./flaskapp \
  -f values-prod.yaml \
  -n flask-prod
```

Verify:

```bash
helm list -n flask-prod
```

The STATUS column must show `deployed`.

### 2.5 Upgrade

```bash
helm upgrade flaskapp ./flaskapp \
  -f values-prod.yaml \
  --set replicaCount=2 \
  -n flask-prod
```

Check that the number of running pods drops to 2:

```bash
kubectl get pods -n flask-prod
```

---

## Part 3: Secure the Pipeline (Day 28)

**Goal**: the Jenkins pipeline must build, scan, and push without ever storing credentials in code or failing silently on vulnerabilities.

### 3.1 Jenkinsfile stage order

```
Checkout → Unit Tests → Build Image → Trivy Scan → Push to ECR → Update Config Repo
```

### 3.2 Unit Tests stage

Include Python dependency scanning alongside unit tests:

```groovy
stage('Unit Tests') {
    steps {
        sh 'pip install -r requirements.txt'
        sh 'python -m pytest tests/'
        sh 'pip install safety && safety check'
    }
}
```

### 3.3 Trivy Scan stage

Use `--exit-code 1` so the pipeline fails if HIGH or CRITICAL vulnerabilities are found:

```groovy
stage('Trivy Scan') {
    steps {
        sh """
            trivy image \
              --exit-code 1 \
              --severity HIGH,CRITICAL \
              --no-progress \
              ${env.ECR_URL}/flaskapp:${env.IMAGE_TAG}
        """
    }
}
```

### 3.4 Credentials

All secrets must be stored in **Jenkins > Manage Credentials** and referenced with `withCredentials`. Nothing is hardcoded in the Jenkinsfile:

```groovy
stage('Push to ECR') {
    steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                          credentialsId: 'aws-ecr-credentials']]) {
            sh """
                aws ecr get-login-password --region us-east-1 \
                  | docker login --username AWS --password-stdin ${env.ECR_URL}
                docker push ${env.ECR_URL}/flaskapp:${env.IMAGE_TAG}
            """
        }
    }
}
```

---

## Part 4: GitOps with ArgoCD (Day 30)

**Goal**: remove Jenkins's ability to talk to the cluster. All cluster changes go through Git.

### 4.1 Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

### 4.2 Create the flaskapp-config repository

Create a public GitHub repository named `flaskapp-config`. Copy the Helm chart from Part 2 into `helm/flaskapp/` in that repository and push it.

### 4.3 Create the ArgoCD Application

```yaml
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flaskapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-github-username>/flaskapp-config
    targetRevision: HEAD
    path: helm/flaskapp
  destination:
    server: https://kubernetes.default.svc
    namespace: flask-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f application.yaml
```

### 4.4 Add the Update Config Repo stage to Jenkinsfile

Add this as the final stage, after the Push to ECR stage:

```groovy
stage('Update Config Repo') {
    steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GIT_TOKEN')]) {
            sh """
                git clone https://${GIT_TOKEN}@github.com/<your-github-username>/flaskapp-config.git
                cd flaskapp-config
                sed -i 's|tag: .*|tag: "${env.IMAGE_TAG}"|' helm/flaskapp/values.yaml
                git config user.email "ci@yourcompany.com"
                git config user.name "Jenkins CI"
                git add helm/flaskapp/values.yaml
                git commit -m "chore: update flaskapp image to ${env.IMAGE_TAG}"
                git push
            """
        }
    }
}
```

### 4.5 End-to-end verification

1. Push a change to the `flaskapp` code repository.
2. Confirm that Jenkins runs all stages and completes successfully, including the Trivy scan.
3. Confirm that a new commit appears in the `flaskapp-config` repository with an updated image tag.
4. Watch the ArgoCD UI: within 3 minutes (or trigger with `argocd app sync flaskapp`) the application should move from **OutOfSync** to **Healthy**.
5. Confirm the new image tag is running: `kubectl get pods -n flask-prod -o wide`.

---

## Part 5: Graduation Checklist

Work through this checklist honestly. Every item should be something you can do from memory or with minimal reference, not just something you have seen once.

### Week 1 — Linux and Git

- [ ] Can navigate the Linux filesystem, manage file permissions, and use vim or nano
- [ ] Understand process management (ps, kill, systemd)
- [ ] Can set up SSH key authentication
- [ ] Can create branches, commits, and pull requests in Git and GitHub

### Week 2 — Docker

- [ ] Can write a multi-stage Dockerfile with a non-root USER and a HEALTHCHECK
- [ ] Understand image layer caching and how to structure a Dockerfile to use it
- [ ] Can use Docker Compose to run multi-container applications locally
- [ ] Can push and pull images to and from ECR

### Week 3 — Kubernetes and Jenkins

- [ ] Can deploy a Deployment, Service, ConfigMap, and Ingress in Kubernetes
- [ ] Understand the difference between ClusterIP, NodePort, and LoadBalancer
- [ ] Can write a declarative Jenkinsfile with multiple stages
- [ ] Can set up a GitHub webhook to trigger Jenkins builds automatically

### Week 4 — AWS and Terraform

- [ ] Can create IAM users, roles, and policies following least privilege
- [ ] Can design a VPC with public and private subnets
- [ ] Can provision EC2, ALB, ASG, and RDS with Terraform
- [ ] Can manage Terraform state in S3 with DynamoDB locking

### Week 5 — Observability

- [ ] Can run a Prometheus and Grafana stack and create a dashboard
- [ ] Can query CloudWatch Logs with CloudWatch Logs Insights
- [ ] Can create a CloudWatch alarm with an SNS email notification
- [ ] Understand the four golden signals (latency, traffic, errors, saturation)

### Week 6 — Production Readiness

- [ ] Can set up Kubernetes RBAC with least-privilege ServiceAccounts
- [ ] Can package a Kubernetes application as a Helm chart with multiple environments
- [ ] Can run Trivy image scans and fail CI on critical vulnerabilities
- [ ] Can diagnose CrashLoopBackOff, Pending, and Service routing issues
- [ ] Can implement a blue-green deployment
- [ ] Can deploy an application via ArgoCD with auto-sync and self-heal enabled
