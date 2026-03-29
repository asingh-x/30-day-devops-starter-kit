# Full Curriculum — DevOps Starter Kit

6 weeks. 30 days. ~5 hours/day. One goal: get you ready for a Level 1 DevOps role.

---

## Week 1 — Linux + Git

**Goal:** Be comfortable working in a Linux terminal and managing code with Git.

| Day | Topic | Key Skills |
|-----|-------|------------|
| Day 1 | Linux Basics | Navigate the filesystem, basic commands, man pages |
| Day 2 | Files, Permissions, Users | chmod, chown, users/groups, sudo |
| Day 3 | Processes, Networking, SSH | ps, top, netstat, ssh key setup, sshd config |
| Day 4 | Git Basics | init, add, commit, branches, .gitignore |
| Day 5 | GitHub + Collaboration | Pull requests, merge strategies, tags, commit messages |

**Mini Project:** Set up a Linux VM, create a script that checks disk/memory usage, push it to GitHub with a proper README.

> **Reference:** See [resources/programming-languages.md](../resources/programming-languages.md) for a guide on Bash, Python, Go, YAML, and JSON — the languages you will use throughout this programme.

---

## Week 2 — Docker + Application Build

**Goal:** Package any application as a container and run it anywhere.

| Day | Topic | Key Skills |
|-----|-------|------------|
| Day 6 | What is Docker? | Containers vs VMs, Docker architecture, install |
| Day 7 | Images and Containers | pull, run, exec, logs, inspect, stop, rm |
| Day 8 | Writing Dockerfiles | FROM, RUN, COPY, CMD, ENTRYPOINT, multi-stage builds |
| Day 9 | Networking + Volumes | Bridge/host/none networks, bind mounts, named volumes |
| Day 10 | Docker Compose | Multi-container apps, compose file structure, depends_on |

**Mini Project:** Containerise a simple Python/Node.js web app, push the image to Docker Hub.

---

## Week 3 — Kubernetes + Jenkins CI/CD

**Goal:** Deploy containerised apps to Kubernetes and automate your pipeline with Jenkins.

| Day | Topic | Key Skills |
|-----|-------|------------|
| Day 11 | Kubernetes Concepts | Why K8s, architecture (master/worker), kubectl |
| Day 12 | Pods, Deployments, ReplicaSets | YAML manifests, rolling updates, rollback |
| Day 13 | Services, ConfigMaps, Ingress | ClusterIP, NodePort, environment injection, TLS |
| Day 14 | Jenkins Basics | Install, plugins, jobs, Jenkinsfile, credentials |
| Day 15 | Full CI/CD Pipeline | GitHub → Jenkins → Docker build → push → K8s deploy |

**Mini Project:** Build a complete pipeline: code push triggers build, image push, and Kubernetes deployment.

---

## Week 4 — AWS + Terraform + ECR

**Goal:** Provision real cloud infrastructure as code and deploy a production-grade app.

| Day | Topic | Key Skills |
|-----|-------|------------|
| Day 16 | AWS Basics + IAM | Regions, AZs, IAM users/roles/policies, MFA |
| Day 17 | EC2 + Networking (VPC) | Launch EC2, security groups, VPC, subnets, route tables |
| Day 18 | AWS Storage + Databases | S3, RDS, ALB, ASG, Route 53 DNS |
| Day 19 | Terraform | HCL, providers, resources, state file, remote backend |
| Day 20 | ECR + Container Registry | Private image registry, IAM for ECR, lifecycle policies |

**Capstone Project:** Deploy a Python web app to AWS EC2 behind an ALB, DNS via Route 53, HTTPS via ACM, infrastructure in Terraform, state in S3, images in ECR.

---

## Week 5 — Monitoring + Observability

**Goal:** Instrument your applications so you can see what is happening in production.

| Day | Topic | Key Skills |
|-----|-------|------------|
| Day 21 | Observability Intro | 3 pillars (metrics/logs/traces), SLI/SLO/SLA, golden signals |
| Day 22 | Prometheus + Grafana | Metrics collection, PromQL, dashboards, alerting rules |
| Day 23 | Logging + CloudWatch | Structured logs, CloudWatch Logs, Logs Insights queries |
| Day 24 | CloudWatch Alarms + Alerting | Alarm states, SNS, budgets, alerting best practices |
| Day 25 | Full Observability Stack | Instrument the capstone app end to end |

**Project:** Add Prometheus metrics, CloudWatch logging, and SNS alerting to the Week 4 production deployment.

---

## Week 6 — Security + Helm + GitOps

**Goal:** Harden your deployments and automate production delivery end to end.

| Day | Topic | Key Skills |
|-----|-------|------------|
| Day 26 | Kubernetes RBAC + Secrets | ServiceAccounts, Roles, RoleBindings, SecurityContext, NetworkPolicy |
| Day 27 | Helm | Charts, values, templates, multi-environment deploys, rollback |
| Day 28 | Security Scanning | Trivy image scanning, dependency scanning, secure Jenkinsfile |
| Day 29 | Troubleshooting + Advanced CI/CD | CrashLoopBackOff, debug commands, blue-green, canary |
| Day 30 | GitOps with ArgoCD | Config repo pattern, ArgoCD Application, auto-sync, self-heal |

**Capstone Project:** Deploy the Flask app via ArgoCD with Helm, RBAC, Trivy scanning in the pipeline, and blue-green switchover.

---

## Level 1 DevOps Engineer Skills Checklist

After completing this curriculum you should be able to:

### Linux
- [ ] Navigate and manage files/directories from the command line
- [ ] Understand and change file permissions (chmod, chown)
- [ ] Manage users, groups, and sudo access
- [ ] SSH into remote servers using key-based authentication
- [ ] Use grep, awk, sed, ps, top, netstat for troubleshooting

### Programming + Scripting
- [ ] Write Bash scripts with error handling (`set -euo pipefail`)
- [ ] Automate tasks with Python (file I/O, JSON/YAML, REST APIs)
- [ ] Read and understand Go code (error handling, CLI tools)
- [ ] Use `jq` to parse JSON from AWS CLI output

### Git
- [ ] Create repos, branches, commits, and pull requests
- [ ] Resolve merge conflicts
- [ ] Write clear commit messages and PR descriptions
- [ ] Understand Git workflows (feature branch, trunk-based)

### Docker
- [ ] Write a multi-stage Dockerfile with non-root USER and HEALTHCHECK
- [ ] Build, tag, and push images to ECR
- [ ] Run and debug containers
- [ ] Use Docker Compose for multi-container apps

### Kubernetes
- [ ] Understand Pods, Deployments, Services, ConfigMaps, Secrets
- [ ] Write and apply YAML manifests
- [ ] Perform rolling updates and rollbacks
- [ ] Set up RBAC with least-privilege ServiceAccounts
- [ ] Package and deploy apps using Helm

### CI/CD
- [ ] Set up a Jenkins pipeline from scratch
- [ ] Write a Jenkinsfile with build, test, scan, push, deploy stages
- [ ] Store secrets securely using Jenkins credentials
- [ ] Integrate Trivy image scanning as a pipeline gate

### AWS
- [ ] Launch and connect to EC2 instances
- [ ] Configure VPC, subnets, security groups
- [ ] Use IAM roles and policies correctly
- [ ] Set up S3, RDS, ALB, Route 53, ECR

### Terraform
- [ ] Write Terraform to provision AWS resources
- [ ] Use variables, outputs, and locals
- [ ] Manage state files in S3 with DynamoDB locking

### Observability
- [ ] Run Prometheus + Grafana and create dashboards
- [ ] Query CloudWatch Logs with Logs Insights
- [ ] Create CloudWatch alarms with SNS notifications
- [ ] Understand the four golden signals

### Production Readiness
- [ ] Diagnose CrashLoopBackOff, OOMKilled, and Pending pods
- [ ] Implement blue-green and canary deployments
- [ ] Deploy via ArgoCD with auto-sync and self-heal
- [ ] Scan images and dependencies in CI/CD pipelines

---

## Recommended Daily Routine

```
Morning   (2h)  — Study new concept + take notes
Afternoon (2h)  — Hands-on practice / exercises
Evening   (1h)  — Revision of previous topics (spaced repetition)
```

Revision schedule: review the previous week on Days 6, 11, 16, 21, and 26.
