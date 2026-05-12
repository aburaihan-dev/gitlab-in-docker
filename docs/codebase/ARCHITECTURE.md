# Architecture

## Core Sections (Required)

### 1) Architectural Style

- Primary style: **Instructional / Example-driven reference collection**
- Why this classification: The repo is not a deployable service. It is a progressive series of 11 self-contained modules, each demonstrating a distinct GitLab CI/CD pattern. There are no shared services, no inter-module dependencies, and no application data flow.
- Primary constraints:
  1. Each module must be independently runnable (no shared state)
  2. All infrastructure is local (no cloud required, uses `localhost` URLs throughout)
  3. Target audience is learners — configs are intentionally simplified (e.g., `puma['worker_processes'] = 0` to reduce RAM)

---

### 2) System Flow

Each module follows this general flow when used:

```text
User runs "docker compose up"
  → GitLab CE container starts (port 8000 or 8088)
  → GitLab data persisted to ./gitlab/ host volumes
  → User registers GitLab Runner (manual or auto via module 07)
  → Runner connects to GitLab server
  → User creates a repo in GitLab and adds a .gitlab-ci.yml
  → Pipeline triggers → Runner picks up job → Executor runs job script
  → Results visible in GitLab UI
```

**Module-specific variations:**

| Module | Executor | How jobs run |
|--------|----------|-------------|
| 03 | Shell | Jobs run directly on the runner host OS |
| 04 | Docker (socket binding) | Jobs run in containers via host Docker socket |
| 05 | Docker-in-Docker (DinD) | Jobs run in containers via privileged docker:dind service |
| 06 | Kubernetes | Jobs run as Pods in a local kind cluster |
| 07 | Docker (auto-register) | Runner auto-registers on startup via `docker compose` command |
| 08 | Docker + Kaniko | Kaniko builds Docker images without Docker daemon (rootless) |
| 09–10 | Docker + Registry | Pipeline builds, pushes, and scans images in GitLab Container Registry |
| 11 | Docker | GitLab Dependency Scanning template scans `requirements.txt` |

---

### 3) Layer/Module Responsibilities

| Layer or module | Owns | Must not own | Evidence |
|-----------------|------|--------------|----------|
| `docker-compose.yml` | Service topology, ports, volumes, env vars, networks | CI pipeline logic | All `docker-compose.yml` files |
| `config.toml` | GitLab Runner registration: executor type, token, Docker image, volumes, network | Service startup | `03.*/config.toml` – `10.*/config.toml` |
| `.gitlab-ci.yml` / `gitlab-ci.yml` | Pipeline job definitions: stages, scripts, images | Infrastructure provisioning | All `*gitlab-ci.yml` files |
| `Dockerfile` | Sample application image definition (for demo purposes only) | Business logic | `04.*/Dockerfile`, `05.*/Dockerfile`, `09.*/Dockerfile`, `10.*/Dockerfile` |
| `README.md` | Step-by-step instructional narrative for each module | Config or code | All `README.md` files |

---

### 4) Reused Patterns

| Pattern | Where found | Why it exists |
|---------|-------------|---------------|
| Volume-mounted GitLab data dirs | All modules with `gitlab-server` service | Survive container restarts (`./gitlab/config`, `./gitlab/data`) |
| `network_mode: host` on runner | Modules 03–05, 07–10 | Allows runner to reach `http://localhost:8000` GitLab server |
| Docker socket binding | Modules 04, 07–10 | Runner creates job containers on the host Docker daemon |
| Privileged mode | Module 05 (DinD), Module 06 (k8s) | Required for Docker-in-Docker and some Kubernetes operations |
| `puma['worker_processes'] = 0` | Modules 02 (partial), 10 | Reduces GitLab memory footprint for local demo use |
| `CI_COMMIT_SHA:0:8` image tagging | Modules 09, 10 | Short-SHA tagging for traceability |
| GitLab CI security templates | Modules 10, 11 | `include: template:` pattern for Container and Dependency Scanning |

---

### 5) Known Architectural Risks

- **Credentials hardcoded across all modules**: Root password, runner registration tokens, and actual runner tokens (`glrt-*`) are committed in plain text. Anyone with repo access can authenticate to a running instance. (See CONCERNS.md)
- **Unpinned image tags**: `gitlab/gitlab-ce:latest` and `gitlab/gitlab-runner:alpine` will silently pull new versions, causing reproducibility failures and potential breaking changes.
- **`network_mode: host` is Linux-only**: These compositions will not work on Docker Desktop for Mac or Windows without modification, limiting audience portability.
- **No cross-module progression automation**: Learners must manually reconfigure each module. There is no shared base compose file.

---

### 6) Evidence

- `02. gitlab-in-docker-compose/docker-compose.yaml` (base pattern established)
- `05. gitlab-runner-with-docker-executor-dind/docker-compose.yml` (DinD networking)
- `06. gitlab-runner-with-kubernetes-executor/config.toml` (k8s executor)
- `08. build-docker-images-using-kaniko/.gitlab-ci.yml` (Kaniko pattern)
- `10. scan-container-images-in-registry/.gitlab-ci.yml` (security scanning)
