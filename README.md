# TutorForge — Peer Tutoring & Code Review Platform

**Mono-repo (microservices)** demo project built with Next.js (TypeScript) + Node services. TutorForge connects students with peer tutors and code reviewers and includes a gamified ranking engine (XP, badges, leaderboards). The repo demonstrates a realistic microservices stack and DevOps workflow: Docker, Kubernetes (ArgoCD GitOps), Terraform + Ansible, GH Actions CI, Prometheus/Grafana, Sentry, Redis + BullMQ, Prisma + Postgres.

> Goal: Production-like, interview-friendly project you can run locally, extend, and deploy via GitOps. Use it to show both full-stack and DevOps skills on your CV.

---

## Table of Contents
1. Project overview
2. Architecture (high level)
3. Services
4. Tech stack
5. CI / CD & GitOps (high level)
6. Infrastructure (Terraform & Ansible)
7. Observability & monitoring
8. Backups & disaster recovery

---

## 1) Project overview
TutorForge combines two student-focused ideas: a **Peer Code Review** platform and a **Peer Tutoring Marketplace**. It emphasizes microservices design, async processing, ranking/gamification, and modern DevOps practices.

Core user journeys:
- Students search tutors, book sessions, and leave ratings.
- Tutors accept sessions and perform code reviews.
- Reviewers earn XP for reviews and accepted suggestions; leaderboards show top tutors.

---

## 2) Architecture (high level)
```
Frontend (Next.js App Router)  <-->  BFF / API Gateway
     |                                   |
     |                                   +--> Auth/User Service (Prisma + Postgres)
     |                                   +--> Tutor Profile Service
     |                                   +--> Booking/Session Service
     |                                   +--> Code Review Service
     |                                   +--> Ranking Service
     |                                   +--> Media Service
     |                                   +--> Notification Service (BullMQ + Redis)
     |                                   +--> Search Service (Meilisearch / Postgres FT)
     |
Shared infra: Redis, Object Storage, GHCR, Prometheus, Grafana, Sentry
Deployment: Docker -> GHCR -> Kubernetes (ArgoCD GitOps)
IaC: Terraform (provision) + Ansible (post-provision bootstrapping)
```

---

## 3) Services (mono-repo layout)
```
/apps
  /frontend            # Next.js app (App Router) — public UI + SSR
  /bff                 # BFF or API Gateway
  /user-service        # auth, profiles
  /tutor-service       # tutor onboarding, specialties
  /booking-service     # sessions & bookings
  /review-service      # code submissions + reviews
  /ranking-service     # points engine, badges, leaderboards
  /notification-service# email + scheduled jobs (BullMQ workers)
/libs                  # shared types + helpers
/infra                 # k8s manifests, Helm charts, ArgoCD apps
/terraform             # Terraform modules for cloud infra
/ansible               # Ansible playbooks for bootstrapping
/.github/workflows     # CI workflows
```

Each service is a small Node/TypeScript app with its own Dockerfile and Prisma schema (for MVP you can use a single Postgres instance with separate schemas).

---

## 4) Tech stack (final choices)
- **Frontend:** Next.js (TypeScript, App Router)
- **Styling:** Tailwind CSS + ShadCN UI
- **State / Data fetching:** React Query + Zustand
- **Backend & microservices:** Node.js (TypeScript) + Prisma + Postgres
- **Queues & workers:** Redis + BullMQ
- **Containers:** Docker, registry = GHCR
- **Orchestration:** Kubernetes (DigitalOcean Managed k8s or k3s for small clusters)
- **GitOps:** ArgoCD (auto-sync infra from Git)
- **CI:** GitHub Actions (lint/tests/build/push/update manifests)
- **IaC:** Terraform (provision infra) + Ansible (post-provision config)
- **Observability:** Prometheus + Grafana + Sentry
- **Secrets:** GitHub Secrets + cloud secret manager (DO/AWS Secrets Manager)

---

## 5) CI / CD & GitOps (high-level)
Pipeline (recommended flow):
1. PR opened → GH Actions runs lint + tests (per-service matrix). Required checks before merge.
2. Merge to `main` → GH Actions builds Docker images (per-service), pushes to GHCR.
3. GH Action updates image tags in `infra/` manifests (commit & push to infra branch or repo).
4. **ArgoCD** watches the infra repo/path and auto-syncs changed apps to k8s cluster (per-service ArgoCD Application). Health checks & readiness probes ensure safe rollouts.

**Important GH Secrets:** `GHCR_TOKEN`, `KUBECONFIG` (or use cloud provider OIDC), `TF_VAR_*` for Terraform, SENTRY_DSN (for prod). Do not store secrets in repo.

---

## 6) Infrastructure (Terraform & Ansible)
**Terraform** responsibilities:
- Provision the Kubernetes cluster
- Create Managed Postgres instance
- Create Redis (managed) or provide credentials
- Create object storage (DO Spaces / S3)
- Configure DNS records (Route53 / DO DNS)

**Ansible** responsibilities (post-provision):
- Bootstrap node exporters, configure cloud logging agents
- Deploy any OS-level utilities that Terraform doesn't manage
- Run initial Prometheus/Grafana helm charts if not managed via ArgoCD

Tip: prefer ArgoCD for app-level Helm releases; keep Terraform focused on cloud resources.

---

## 7) Observability & monitoring
- **Sentry**: integrate DSN into each service for error reporting (release tagging helps correlate errors to deploys)
- **Prometheus**: instrument services with `prom-client` (Node). Expose `/metrics` endpoint.
- **Grafana**: create dashboards for request latency, error rate, queue depth (BullMQ), XP events per minute, sessions booked.

Add alerting rules for:
- high API error rate
- queue backlog size
- failed migrations/deploy

---

## 8) Backups & Disaster Recovery
- Enable managed DB daily snapshots and automated retention.
- Add a `scripts/backup-pg.sh` that runs `pg_dump` to object storage; test `docs/restore.md` periodically.
- Document DR runbook under `docs/restore.md` and record a one-time restore test.

---

*Last updated: 2025-09-29*

