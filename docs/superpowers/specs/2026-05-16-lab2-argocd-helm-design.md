# Lab2 Design Spec — ArgoCD & Helm for Zen Pharma Platform

**Date:** 2026-05-16  
**Scope:** lab2/ directory — student hands-on lab for Days 2 and 3  
**Depends on:** lab1/ (namespace, RBAC, secrets must exist)

---

## Context

Lab1 taught raw `kubectl apply` across all 9 services. Students felt the manual pain. Lab2 introduces ArgoCD (Day 2) and then Helm (Day 3) to solve it — using the same services, same cluster, same repo.

The existing `argocd/` and `helm-charts/` directories are the instructor's production reference. Lab2 gives students their own working copies to build up from scratch.

---

## Directory Layout

```
lab2/
├── LAB2-GUIDE.md
├── manifests/                        # Day 2: raw K8s YAMLs per service
│   ├── auth-service/
│   │   ├── serviceaccount.yaml
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── api-gateway/                  # same 4 files
│   ├── drug-catalog-service/
│   ├── inventory-service/
│   ├── manufacturing-service/
│   ├── supplier-service/
│   ├── qc-service/                   # no secrets in envFrom
│   ├── notification-service/         # Node.js, port 3000
│   └── pharma-ui/                    # React/nginx, port 80, extra volume mounts
└── argocd-apps/
    ├── project/
    │   └── pharma-project.yaml       # AppProject — shared by Day 2 and Day 3
    ├── day2-raw/                     # 9 apps pointing to lab2/manifests/<service>/
    │   └── <service>-app.yaml × 9
    └── day3-helm/
        ├── dev/                      # 9 apps → helm-charts + envs/dev/values-<svc>.yaml
        ├── qa/                       # 9 apps → helm-charts + envs/qa/values-<svc>.yaml
        └── prod/                     # 9 apps → helm-charts + envs/prod/values-<svc>.yaml
```

---

## Key Design Decisions

### Day 2: Raw manifests via ArgoCD

- Each `lab2/manifests/<service>/` is an independent ArgoCD source path
- ArgoCD watches the directory and applies all YAMLs in it (no Helm block in app spec)
- Values are hardcoded from `envs/dev/values-*.yaml` — this is intentional; it shows why values files exist
- Secrets (`db-credentials`, `jwt-secret`) are pre-created manually (same as lab1 Step 4)
- Day 2 deploys dev only; qa/prod come in Day 3

### Day 3: Helm migration

- `day3-helm/dev/` app specs are identical in structure to production `argocd/apps/dev/` but with student-facing comments
- `spec.source.path: helm-charts` + `helm.valueFiles: ['../envs/dev/values-<svc>.yaml']`
- No new Helm chart files needed — existing `helm-charts/` and `envs/` are reused
- Student deletes day2-raw apps, applies day3-helm apps — same end state, different source
- qa: automated sync; prod: manual sync only (no `automated:` block)

### ArgoCD Application naming

| Day | App name pattern | Namespace |
|-----|-----------------|-----------|
| Day 2 | `auth-service-dev-raw` | dev |
| Day 3 dev | `auth-service-dev` | dev |
| Day 3 qa | `auth-service-qa` | qa |
| Day 3 prod | `auth-service-prod` | prod |

Distinct names prevent collision when students run Day 3 alongside Day 2 apps.

---

## Service Reference

| Service | Port | Secrets | Notes |
|---------|------|---------|-------|
| auth-service | 8081 | db-credentials, jwt-secret | Anchor service |
| api-gateway | 8080 | db-credentials, jwt-secret | Ingress enabled, IRSA annotation |
| drug-catalog-service | 8082 | db-credentials | Flyway migrations — longer probe delays |
| inventory-service | 8083 | db-credentials | Standard Spring Boot |
| manufacturing-service | 8085 | db-credentials | Standard Spring Boot |
| supplier-service | 8084 | db-credentials | Standard Spring Boot |
| qc-service | 8086 | none | No DB secrets |
| notification-service | 3000 | db-credentials | Node.js — different probe paths/delays |
| pharma-ui | 80 | none | React/nginx — 3 emptyDir volumes needed |

---

## LAB2-GUIDE.md Flow

### Day 2

1. Prerequisites check (EKS up, kubectl configured, lab1 done)
2. Install ArgoCD (`kubectl apply -n argocd`)
3. Access UI (port-forward) + CLI login
4. Understand ArgoCD concepts: AppProject, Application, sync, reconciliation loop
5. Create pharma AppProject
6. Apply `day2-raw/auth-service-app.yaml` — watch sync of raw manifests
7. ArgoCD UI deep-dive: resource tree, diff, history, logs, drift simulation
8. Scale to all 9 services (`for app in lab2/argocd-apps/day2-raw/*.yaml`)
9. Pain point: count the YAML files, try updating an image tag across all

### Day 3

1. Recap pain points from Day 2
2. Helm chart structure walkthrough (`helm-charts/` directory)
3. `helm template auth-service ./helm-charts -f envs/dev/values-auth-service.yaml`
4. Delete Day 2 auth-service app, apply `day3-helm/dev/auth-service-app.yaml`
5. Observe: same K8s resources, ArgoCD now renders via Helm
6. Apply all 9 dev services — one chart, 9 values files
7. Deploy qa (`day3-helm/qa/`) — auto-sync
8. Deploy prod (`day3-helm/prod/`) — manual sync only, show gate
9. Simulate CI: `yq e '.image.tag = "sha-demo123"' -i envs/dev/values-auth-service.yaml` → commit → ArgoCD auto-syncs dev
