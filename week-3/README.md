# Week 3 — Kubernetes + Jenkins CI/CD

**Goal:** Deploy containerized apps to Kubernetes and automate your pipeline with Jenkins.

---

## Days

| Day | File | Topic |
|-----|------|-------|
| Day 11 | [day-11-kubernetes-concepts.md](./day-11-kubernetes-concepts.md) | Why K8s, architecture, kubectl basics |
| Day 12 | [day-12-pods-deployments.md](./day-12-pods-deployments.md) | Pods, Deployments, ReplicaSets, rolling updates |
| Day 13 | [day-13-services-configmaps-ingress.md](./day-13-services-configmaps-ingress.md) | Services, ConfigMaps, Secrets, Ingress |
| Day 14 | [day-14-jenkins-basics.md](./day-14-jenkins-basics.md) | Jenkins install, jobs, Jenkinsfile, credentials |
| Day 15 | [day-15-cicd-pipeline.md](./day-15-cicd-pipeline.md) | Full pipeline: GitHub → build → push → deploy |

## Mini Project

[exercises.md](./exercises.md) — Build a complete CI/CD pipeline that deploys to Kubernetes on every code push.

## What You Should Know After This Week

- Explain the Kubernetes architecture (master/worker, API server, etcd, scheduler)
- Write YAML manifests for Pods, Deployments, and Services
- Understand the difference between ClusterIP, NodePort, and LoadBalancer
- Set up a Jenkins pipeline with a Jenkinsfile
- Automate build → push → deploy from a GitHub trigger
