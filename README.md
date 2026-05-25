# DevSecOps MERN Production Pipeline

A production-grade DevSecOps implementation featuring dual CI/CD pipelines, GitOps-driven Kubernetes deployment, container security enforcement, and full observability — built for a MERN stack application.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Frontend Pipeline](#frontend-pipeline)
- [Backend Pipeline](#backend-pipeline)
- [GitOps Deployment](#gitops-deployment)
- [Security Layer](#security-layer)
- [Kubernetes Orchestration](#kubernetes-orchestration)
- [Observability Stack](#observability-stack)
- [Branch Strategy](#branch-strategy)
- [Tech Stack](#tech-stack)
- [Future Roadmap — AWS Scaling](#future-roadmap--aws-scaling)

---

## Overview

This project implements a **dual-pipeline DevSecOps architecture** for a full-stack MERN application. The system separates concerns cleanly: the frontend and backend have independent CI/CD pipelines, each optimised for its deployment target. Security is enforced at every stage — not bolted on at the end.

```
GitHub Repository
       │
   ┌───┴────┐
   │        │
Frontend  Backend
  CI/CD    CI/CD
(Actions) (Jenkins)
   │        │
Vercel   Docker Hub
            │
         ArgoCD (GitOps)
            │
      Kubernetes Cluster
      (HPA · Sealed Secrets)
            │
   Prometheus + Grafana
```

---

## Architecture

The project is organised into two folders — `client/` (React frontend) and `backend/` (Node/Express API) — each with its own dedicated pipeline.

### Why two separate pipelines?

Frontend and backend have fundamentally different deployment targets, risk profiles, and tooling requirements. Coupling them in a single pipeline would create unnecessary bottlenecks and complicate rollback strategies. Keeping them independent means a frontend UI change can ship without touching the backend pipeline, and a backend security patch can be deployed without a full frontend rebuild.

---

## Frontend Pipeline

**Toolchain:** GitHub Actions → Vercel

### Why GitHub Actions?

GitHub Actions is native to the repository — no external CI server to maintain, no webhook plumbing required. It integrates directly with branch protection rules, pull request checks, and Dependabot, making it the natural choice for frontend CI.

### Why Vercel?

Vercel provides a globally distributed CDN, DDoS protection, and automatic preview deployments — all on a free tier. For a project of this scale, that's the right trade-off: zero infrastructure overhead for the frontend, with production-grade reliability built in.

### Pipeline Steps

| Step | Tool | Purpose |
|------|------|---------|
| Unit tests | Vitest | Component and logic validation |
| Lint + format | ESLint + Prettier | Code quality enforcement |
| Performance audit | Google Lighthouse | Core Web Vitals tracking |
| Security scan | CodeQL | Static analysis for vulnerabilities |
| Dependency scan | Dependabot | Automated CVE detection in npm packages |
| Deploy | Vercel | Edge deployment, only on passing checks |

### Branch Protection

The `main` branch is protected: all status checks (CI + Vercel) must pass before a merge is allowed. Deployments to production only happen through this gate.

---

## Backend Pipeline

**Toolchain:** Jenkins → Docker → Docker Hub

### Why Jenkins?

Jenkins is open-source, self-hosted, and highly customisable — the industry standard for organisations that need full control over their CI environment. It was chosen here to demonstrate real-world pipeline design outside of managed CI offerings.

### Pipeline Steps

**1. Docker multi-stage build**
The Dockerfile uses multi-stage builds to separate the build environment from the runtime image. The final production image contains only what is strictly necessary — no dev dependencies, no build tools. This reduces image size and attack surface.

**2. Hardened image**
The production image is configured with a non-root user, read-only filesystem where possible, and minimal base layer. This follows container security best practices enforced at build time, not as an afterthought.

**3. SonarQube analysis**
Code quality and test coverage are measured by SonarQube. Quality gates are configured: the pipeline fails if coverage drops below threshold or if critical code smells are introduced.

**4. Trivy CVE scanning**
Before the image is pushed to any registry, Trivy scans it for known CVEs. **The image is only pushed and deployed if this scan passes.** This is the hard security gate in the backend pipeline.

**5. Push to Docker Hub**
Passing images are tagged with a version identifier and pushed to Docker Hub. Versioned tags — not `latest` — are used throughout, enabling precise rollbacks.

---

## GitOps Deployment

**Toolchain:** ArgoCD + Helm

A separate `devsecops` branch in the repository holds all Kubernetes manifests and Helm charts. ArgoCD watches this branch continuously and automatically syncs the cluster state when changes are detected.

When Jenkins pushes a new image tag to Docker Hub, the image version in the Helm chart is updated. ArgoCD detects this change and triggers a rolling deployment — no manual `kubectl apply` required.

This decouples deployment from CI: the cluster's desired state is always described in Git, and ArgoCD is the reconciliation engine that enforces it.

---

## Security Layer

Security is enforced across the entire delivery chain, not just at one point:

| Control | Where | What it does |
|---------|-------|-------------|
| CodeQL | GitHub Actions (frontend) | Static analysis for vulnerabilities in JS/TS code |
| Dependabot | GitHub (both) | Automated PRs for vulnerable dependency versions |
| SonarQube | Jenkins (backend) | Code quality gates and coverage enforcement |
| Trivy | Jenkins (backend) | CVE scanning before image push — hard deploy gate |
| Sealed Secrets | Kubernetes | Encrypted secrets committed safely to Git |
| Branch protection | GitHub | No direct pushes to `main`; status checks required |

### Sealed Secrets

Kubernetes Secrets are encrypted at rest using the Sealed Secrets controller. The encrypted `SealedSecret` resource can be safely committed to the `devsecops` branch — only the controller inside the cluster can decrypt it. This solves the classic problem of secret management in a GitOps workflow.

---

## Kubernetes Orchestration

The backend runs on Kubernetes with the following configuration:

**Horizontal Pod Autoscaler (HPA)**
HPA monitors CPU and memory utilisation and scales the number of backend pods up or down automatically. This handles traffic spikes without manual intervention.

**Minimum replica guarantee**
A minimum of two replicas is enforced at all times. This ensures the service remains available during rolling deployments and single-node failures — one pod is always serving traffic while the other is being updated.

**Load-balanced services**
A Kubernetes Service resource distributes incoming traffic across all healthy pods. Combined with HPA, this provides elastic, fault-tolerant scaling.

---

## Observability Stack

| Tool | Role |
|------|------|
| Prometheus | Metrics scraping and storage |
| Grafana | Dashboard visualisation and alerting |
| Loki *(planned)* | Centralised log aggregation |

Prometheus scrapes metrics from the backend pods and Kubernetes components. Grafana dashboards visualise request rates, error rates, latency percentiles, and pod resource usage. Loki integration is planned to bring structured log search into the same Grafana interface.

---

## Branch Strategy

| Branch | Purpose |
|--------|---------|
| `main` | Production-ready application code |
| `ci` | Frontend CI/CD GitHub Actions workflows |
| `jenkins-ci` | Backend Jenkins pipeline configuration |
| `devsecops` | Kubernetes manifests, Helm charts, ArgoCD config |

---

## Tech Stack

### Application
- **MongoDB** — document database
- **Express.js** — Node.js web framework
- **React.js** — frontend UI library
- **Node.js** — JavaScript runtime

### CI/CD
- **GitHub Actions** — frontend CI automation
- **Jenkins** — backend CI automation
- **Docker** — containerisation with multi-stage builds
- **Vercel** — frontend hosting and CDN

### GitOps & Orchestration
- **Kubernetes** — container orchestration
- **ArgoCD** — GitOps continuous deployment
- **Helm** — Kubernetes package management
- **Docker Hub** — container image registry

### Security
- **SonarQube** — code quality and coverage gates
- **Trivy** — container CVE scanning
- **CodeQL** — static application security testing
- **Sealed Secrets** — encrypted secret management for Kubernetes

### Observability
- **Prometheus** — metrics collection
- **Grafana** — dashboards and alerting
- **Loki** *(planned)* — log aggregation

---

## Future Roadmap — AWS Scaling

This system is designed to be cloud-native. The following migration path maps each component to its AWS equivalent, preserving the architectural patterns while gaining managed infrastructure.

### Compute
- **Current:** Self-managed Kubernetes cluster
- **AWS:** Amazon EKS with managed node groups and Cluster Autoscaler

### Container Registry
- **Current:** Docker Hub
- **AWS:** Amazon ECR — private registry with built-in image scanning, IAM-based access control, and lifecycle policies

### CI/CD
- **Current:** Jenkins
- **AWS:** AWS CodePipeline + CodeBuild, or a hybrid model keeping Jenkins with AWS deployment targets

### Database
- **Current:** MongoDB (self-hosted)
- **AWS:** Amazon DocumentDB (MongoDB-compatible) or MongoDB Atlas on AWS, with EBS-backed persistent volumes for stateful workloads

### Networking & Security
- **Current:** Kubernetes ingress controller
- **AWS:** Application Load Balancer (ALB) with AWS WAF, Route 53 for DNS management and health-check routing

### Secrets Management
- **Current:** Sealed Secrets
- **AWS:** AWS Secrets Manager — fully managed, with automatic rotation and fine-grained IAM policies

### Observability
- **Current:** Prometheus + Grafana + Loki
- **AWS:** Amazon CloudWatch for logs and metrics, AWS X-Ray for distributed tracing

### Cost & Scaling Model

The current self-hosted architecture is optimised for learning and demonstration. The AWS roadmap is designed for production multi-team environments where managed services reduce operational overhead at the cost of increased spend. The architectural patterns — GitOps, immutable images, policy-gated deployments, horizontal scaling — carry over directly.

---

## Engineering Skills Demonstrated

- **CI/CD pipeline design** — dual-pipeline architecture across two different CI systems
- **GitOps** — declarative infrastructure management with ArgoCD and Helm
- **Container security** — multi-stage hardened builds, CVE scanning as a deploy gate, image signing
- **Kubernetes orchestration** — HPA, replica guarantees, rolling deployments
- **Secrets management** — encrypted secrets committed safely to Git via Sealed Secrets
- **Observability** — metrics, dashboards, and structured logging
- **Scalable system design** — architecture designed for cloud-native extension to AWS
