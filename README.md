# CI/CD Pipeline — FastAPI Microservices on Kubernetes

> Production-grade CI/CD pipeline built with Jenkins, Kubernetes (K3s), Helm, and Docker on AWS EC2.  
> Two FastAPI microservices deployed across 4 isolated environments with full automation from commit to staging, and gated manual promotion to production.

---

## Architecture Overview

```
GitHub (push)
     │
     ▼
Jenkins Pipeline
     │
     ├── Checkout → detect changed services (git diff)
     ├── Build & Push → Docker Hub (tagged by Git commit SHA)
     │
     ├── [branch: dev] ──────────────────────────────────────────┐
     │        ├── Deploy dev    (NodePort — debug access)         │
     │        ├── Test dev      (helm test + kubectl wait)        │
     │        ├── Deploy qa     (NodePort)                        │
     │        ├── Test qa                                         │
     │        ├── Deploy staging (ClusterIP — Ingress only)       │
     │        ├── Test staging  (helm test + curl external)       │
     │        └── Save image tags → Git commit [skip ci]          │
     │                                                            │
     └── [branch: master] ─────────────────────────────────────┐ │
              ├── Sanity Check  (manual gate — input prompt)    │ │
              ├── Deploy prod   (ClusterIP — Ingress only)      │ │
              └── Test prod     (helm test + curl external)     │ │
                                                                ▼ ▼
                                         K3s Cluster (AWS EC2 Ubuntu)
                                              │
                                    ┌─────────┴─────────┐
                               Traefik Ingress Controller
                                    │                   │
                              /api/v1/movies      /api/v1/casts
                                    │                   │
                             movie-service-svc   cast-service-svc
                                    │                   │
                               Pods (FastAPI)     Pods (FastAPI)
                                    │                   │
                            PostgreSQL (StatefulSet) PostgreSQL (StatefulSet)
```

---

## Stack

| Tool | Role |
|---|---|
| **FastAPI** | REST microservices (`movie-service`, `cast-service`) |
| **PostgreSQL** | Dedicated database per service (StatefulSet) |
| **Docker / Docker Hub** | Image build & registry (`reasg/`) |
| **Helm** | Kubernetes packaging — 1 shared chart, 2 releases |
| **K3s** | Lightweight Kubernetes cluster on AWS EC2 |
| **Jenkins** | CI/CD orchestration (Declarative Pipeline) |
| **Traefik** | Ingress controller (built into K3s) |
| **Nginx** | Reverse proxy for local docker-compose dev only |

---

## Repository Structure

```
Jenkins_for_devops/
├── Jenkinsfile                   # Declarative pipeline (all stages)
├── docker-compose.yml            # Local development environment
├── nginx_config.conf             # Nginx config (docker-compose only)
├── movie-service/                # FastAPI microservice — movies
│   ├── Dockerfile
│   └── app/
├── cast-service/                 # FastAPI microservice — cast
│   ├── Dockerfile
│   └── app/
├── helm/
│   ├── charts/                   # Shared Helm chart (1 chart → 2 releases)
│   │   ├── Chart.yaml
│   │   ├── values.yaml           # Default values
│   │   └── templates/            # Deployment, Service, StatefulSet, Ingress...
│   ├── values-movie.yaml         # movie-service overrides
│   ├── values-cast.yaml          # cast-service overrides
│   ├── values-movie-secret.yaml  # PostgreSQL secret — gitignored
│   ├── values-cast-secret.yaml   # PostgreSQL secret — gitignored
│   └── values-staging-prod.yaml  # ClusterIP override for staging/prod
├── k8s/
│   └── ingress.yaml              # Standalone Ingress (kubectl apply)
└── tags/
    ├── .movie-image-tag          # Last stable image tag (committed by Jenkins)
    └── .cast-image-tag
```

---

## Key Design Decisions

### Smart change detection
The pipeline uses `git diff --name-only HEAD~1 HEAD` to detect which service changed. Only the affected service gets rebuilt and redeployed — the other one uses its last stable image tag (stored in `tags/`). This avoids unnecessary builds and matches how real teams operate.

### Image tagging strategy
Images are tagged with the Git commit SHA (`env.GIT_COMMIT`), not `latest`. This ensures every deployment is fully traceable and reproducible. The `latest` tag is never used in Kubernetes.

### 1 Helm chart, 2 releases
Both microservices share a single parametric Helm chart. Environment-specific config (image, ports, DB credentials, service type) is injected via separate `values-*.yaml` files. This avoids duplication while keeping services independently deployable.

### Service type per environment

| Namespace | Service type | Access |
|---|---|---|
| `dev` | NodePort | Direct port access (30007/30008) + Ingress |
| `qa` | NodePort | Direct port access (30009/30010) + Ingress |
| `staging` | ClusterIP | Ingress only |
| `prod` | ClusterIP | Ingress only |

NodePort in dev/qa allows testing individual services without going through the Ingress — useful for isolation debugging.

### Ingress managed outside Helm
The Ingress resource is applied via `kubectl apply` rather than rendered by Helm. This avoids re-creating it on every `helm upgrade` and keeps routing config stable across deployments.

### Production gate
The `master` branch triggers a `input` step (Sanity Check) before deploying to prod. No code reaches production without explicit human approval — a standard pattern in regulated or critical environments.

### Secrets never committed
PostgreSQL passwords are passed as Helm `--values` files injected by Jenkins credentials (Secret file type). They are `.gitignore`d and never appear in the repository or pipeline logs.

---

## Pipeline Stages

```
Checkout → Build & Push → Deploy dev → Test dev
        → Deploy qa → Test qa
        → Deploy staging → Test staging (helm + curl)
        → Save Tags → [master only] Sanity Check
        → Deploy prod → Test prod (helm + curl)
```

Each test stage combines:
- `kubectl wait --for=condition=Available` — waits for the deployment to be ready
- `helm test` — runs the built-in Helm connectivity test pod
- `curl` external check (staging/prod) — validates the public endpoint end-to-end

---

## Running Locally

```bash
docker-compose up -d
```

| Endpoint | URL |
|---|---|
| Movie service docs | http://localhost:8080/api/v1/movies/docs |
| Cast service docs | http://localhost:8080/api/v1/casts/docs |

---

## Infrastructure Setup (one-time)

### 1. Kubernetes namespaces

```bash
kubectl create namespace dev
kubectl create namespace qa
kubectl create namespace staging
kubectl create namespace prod
```

### 2. Jenkins credentials required

| ID | Type | Used for |
|---|---|---|
| `dockerhub` | Username/Password | Docker Hub push |
| `kubeconfig` | Secret file | Helm → K3s access |
| `values-movie-secret` | Secret file | PostgreSQL password (movie) |
| `values-cast-secret` | Secret file | PostgreSQL password (cast) |
| `GITHUB_CREDS` | Username/Password | Git push image tags |

### 3. Secret file structure

```yaml
# values-movie-secret.yaml  ← never commit this
secret:
  POSTGRES_PASSWORD: yourpassword
```

---

## Manual Deployment

### Dev / QA

```bash
helm upgrade --install movie-service ./helm/charts \
  -f helm/values-movie.yaml \
  -f helm/values-movie-secret.yaml \
  -n dev

helm upgrade --install cast-service ./helm/charts \
  -f helm/values-cast.yaml \
  -f helm/values-cast-secret.yaml \
  -n dev

kubectl apply -f k8s/ingress.yaml -n dev
```

### Staging / Prod

```bash
helm upgrade --install movie-service ./helm/charts \
  -f helm/values-movie.yaml \
  -f helm/values-movie-secret.yaml \
  -f helm/values-staging-prod.yaml \
  -n staging

kubectl apply -f k8s/ingress.yaml -n staging
```

---

## Docker Hub

Images: https://hub.docker.com/repositories/reasg

| Image | Description |
|---|---|
| `reasg/jenkins_for_devops_movie_service` | FastAPI movie microservice |
| `reasg/jenkins_for_devops_cast_service` | FastAPI cast microservice |

Tags follow Git commit SHA for full traceability.

---

## What I Built & Why

This project covers the full DevOps lifecycle from a single `git push`:

- **Containerization** — Dockerfiles per service, multi-stage builds considered
- **Kubernetes packaging** — reusable Helm chart with environment-driven values
- **CI/CD automation** — Jenkins Declarative Pipeline with conditional stage execution
- **Environment promotion** — dev → qa → staging (automatic) → prod (manual gate)
- **Observability basics** — deployment readiness checks + external endpoint validation
- **Secret management** — credentials injected at runtime, never stored in VCS

> Next steps: Terraform for infrastructure provisioning, Ansible for VM configuration, Prometheus + Grafana for metrics.