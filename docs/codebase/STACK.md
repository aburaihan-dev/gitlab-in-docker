# Technology Stack

## Core Sections (Required)

### 1) Runtime Summary

| Area | Value | Evidence |
|------|-------|----------|
| Primary language | YAML (config) / Shell (CI scripts) | All `*.yml`, `*.yaml`, `*.toml` files |
| Runtime | Docker Engine + Docker Compose | All `docker-compose.yml` files |
| Package manager | None (no app code) | — |
| Module/build system | None | — |

> This is a **tutorial/reference repository**, not an application. There is no application runtime. The "stack" is the infrastructure toolchain used to demonstrate GitLab CI/CD patterns.

---

### 2) Production Frameworks and Dependencies

| Dependency | Version | Role in system | Evidence |
|------------|---------|----------------|----------|
| gitlab/gitlab-ce | `latest` (unpinned) | Self-hosted GitLab server | All `docker-compose.yml` files |
| gitlab/gitlab-runner | `alpine` / `latest` | GitLab CI runner | All `docker-compose.yml` files |
| docker:dind | `27.2.0` | Docker-in-Docker service for pipeline jobs | `05.*/gitlab-ci.yml` |
| gcr.io/kaniko-project/executor | `v1.23.2-debug` | Rootless Docker image builder in CI | `08.*/.gitlab-ci.yml` |
| python | `3.10-alpine`, `3.9-alpine` | Base image for sample Dockerfiles and jobs | `04.*/Dockerfile`, `05.*/Dockerfile`, `08.*/src/mydockerfile` |
| kind (Kubernetes in Docker) | unspecified | Local Kubernetes cluster for k8s executor demo | `06.*/kind-cluster-config.yaml` |

---

### 3) Development Toolchain

| Tool | Purpose | Evidence |
|------|---------|----------|
| Docker Desktop / Docker Engine | Run all containers locally | All `docker-compose.yml` files |
| Docker Compose | Orchestrate multi-container GitLab stacks | All `docker-compose.yml` files |
| kind | Create local Kubernetes cluster | `06.*/kind-cluster-config.yaml` |
| kubectl | Apply Kubernetes manifests | `06.*/README.md` |
| jq | Parse CI JSON security reports | `10.*/.gitlab-ci.yml`, `11.*/.gitlab-ci.yml` |

---

### 4) Key Commands

```bash
# Start a GitLab stack (run from inside the numbered module folder)
docker compose up

# Stop stack
docker compose stop

# Destroy stack (removes containers, not volumes)
docker compose down

# Register a GitLab runner (example — see each module's README)
gitlab-runner register --url http://localhost:8000 --token <token>

# Create a kind Kubernetes cluster (module 06)
kind create cluster --config kind-cluster-config.yaml
kubectl apply -f kind-service.yaml
```

---

### 5) Environment and Config

- Config sources: `docker-compose.yml` files per module; `config.toml` per runner module
- Required env vars: None are injected via `.env` files. Credentials are **hardcoded directly** in compose files (see CONCERNS.md)
- GitLab CI variables used at pipeline runtime: `CI_REGISTRY`, `CI_REGISTRY_IMAGE`, `CI_REGISTRY_USER`, `CI_REGISTRY_PASSWORD`, `CI_COMMIT_SHA`, `CI_PROJECT_DIR`, `CI_COMMIT_TAG`
- Deployment/runtime constraints: Requires Linux Docker host (socket binding `/var/run/docker.sock`); some modules require privileged mode

---

### 6) Evidence

- `02. gitlab-in-docker-compose/docker-compose.yaml`
- `03. gitlab-runner-with-shell-executor/docker-compose.yml`
- `05. gitlab-runner-with-docker-executor-dind/docker-compose.yml`
- `06. gitlab-runner-with-kubernetes-executor/config.toml`
- `08. build-docker-images-using-kaniko/.gitlab-ci.yml`
