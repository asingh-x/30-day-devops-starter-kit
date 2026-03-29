# Week 6 — Production Readiness

## What This Week Covers

Week 6 turns a working Kubernetes deployment into one that is secure, auditable, and ready for production traffic. The focus shifts from "does it run?" to "is it safe, repeatable, and observable when things go wrong?"

Topics: Kubernetes RBAC, Secrets management, Helm, container image security scanning, advanced troubleshooting, disaster recovery, cost awareness, and GitOps with ArgoCD.

## Prerequisites

Weeks 1 through 5 completed:

- Week 1: Linux and Git
- Week 2: Docker
- Week 3: Kubernetes and Jenkins
- Week 4: AWS and Terraform
- Week 5: Observability

## What You Will Build

By the end of Week 6, the Flask application from Week 4 will be:

- Running in a dedicated namespace with a least-privilege ServiceAccount and SecurityContext
- Packaged as a Helm chart with separate values files for staging and production
- Scanned for vulnerabilities in CI with Trivy before any image is pushed
- Deployed entirely through GitOps — no `kubectl apply` from Jenkins or a laptop
- Managed by ArgoCD with automatic sync and self-heal enabled

## Day-by-Day

| Day | Topic | File |
|-----|-------|------|
| 26 | Kubernetes RBAC and Secrets | [day-26-kubernetes-rbac-secrets.md](day-26-kubernetes-rbac-secrets.md) |
| 27 | Helm | [day-27-helm.md](day-27-helm.md) |
| 28 | Security Scanning | [day-28-security-scanning.md](day-28-security-scanning.md) |
| 29 | Troubleshooting and Advanced CI/CD | [day-29-troubleshooting-advanced-cicd.md](day-29-troubleshooting-advanced-cicd.md) |
| 30 | GitOps and ArgoCD | [day-30-gitops-argocd.md](day-30-gitops-argocd.md) |

Day 30 is the final day of the full 30-day program.

## Capstone

[exercises.md](exercises.md) contains the Week 6 capstone exercise. It walks through all five parts — RBAC, Helm, pipeline security, GitOps, and a full graduation checklist covering all 6 weeks.

Complete the checklist at the end before considering the program done.
