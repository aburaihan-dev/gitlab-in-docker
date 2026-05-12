# GitLab Runner Setup Guide
**GitLab Version:** 18.11 (CE Edition)  
**Runner Version:** 18.11.3  
**Date:** 2026-05-12  
**Use Case:** Build Docker images + Deploy to remote servers via SSH

---

## Table of Contents

1. [Repository Reference](#1-repository-reference)
2. [Recommended Runner Configuration](#2-recommended-runner-configuration)
3. [docker-compose.yml](#3-docker-composeyml)
4. [config.toml](#4-configtoml)
5. [Runner Registration (GitLab 18.x)](#5-runner-registration-gitlab-18x)
6. [GitLab CI Pipeline Template](#6-gitlab-ci-pipeline-template)
7. [Deployment Approval Strategy (CE Edition)](#7-deployment-approval-strategy-ce-edition)
8. [Healthy Startup Log Reference](#8-healthy-startup-log-reference)
9. [Security Notes](#9-security-notes)

---

## 1. Repository Reference

**Repo:** `aburaihan-dev/gitlab-in-docker`  
**Module used as base:** `04. gitlab-runner-with-docker-executor-socket-binding`

### Why this module?

| Requirement | Coverage |
|---|---|
| Build Docker images | ✅ Socket binding gives jobs access to host Docker daemon |
| SSH to remote servers | ✅ Any job image with `openssh-client` works |
| Runner runs in Docker | ✅ Runner itself is a Docker container |
| No privileged mode needed | ✅ Socket binding avoids `privileged: true` on runner |

### Modules NOT used and why

| Module | Reason Skipped |
|---|---|
| 03 - Shell executor | No isolation — jobs run directly on host OS |
| 05 - Docker-in-Docker | Requires `privileged: true`, slower, more complex |
| 06 - Kubernetes executor | No Kubernetes cluster in scope |
| 07 - Auto-register | **Broken on GitLab 16+** — `GITLAB_SHARED_RUNNERS_REGISTRATION_TOKEN` removed |
| 08 - Kaniko | Rootless image building only — no SSH deployment |

---

## 2. Recommended Runner Configuration

**Executor:** Docker with socket binding  
**Key design decisions:**
- Docker socket (`/var/run/docker.sock`) mounted into runner for image builds
- `network_mode: host` so jobs can reach the GitLab server at `localhost`
- `privileged: false` — socket binding does not need it
- `concurrent: 4` — tune based on VM CPU/RAM
- `pull_policy: if-not-present` — avoids re-pulling cached images

---

## 3. docker-compose.yml

```yaml
services:
  gitlab-runner:
    image: gitlab/gitlab-runner:v18.11.3
    container_name: gitlab-runner
    restart: always
    volumes:
      - ./config:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock
```

> **Note:** Pin the image version (`v18.11.3`) to match your GitLab server — avoids silent breaking changes from `:latest`.

**Directory layout:**
```
your-runner-dir/
├── docker-compose.yml
└── config/
    └── config.toml
```

---

## 4. config.toml

```toml
concurrent = 4
check_interval = 0
connection_max_age = "15m0s"
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "docker-runner"
  url = "http://<YOUR_GITLAB_VM_IP_OR_DOMAIN>"
  token = "glrt-<YOUR_TOKEN_FROM_GITLAB_UI>"
  executor = "docker"

  [runners.custom_build_dir]

  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]

  [runners.docker]
    image = "docker:27.5"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    network_mode = "host"

    volumes = [
      "/var/run/docker.sock:/var/run/docker.sock",
      "/cache"
    ]

    shm_size = 0
    network_mtu = 0
    pull_policy = ["if-not-present"]
```

### config.toml settings explained

| Setting | Value | Reason |
|---|---|---|
| `concurrent` | `4` | Max parallel jobs — tune to VM resources |
| `executor` | `docker` | Each job runs in its own isolated container |
| `image` | `docker:27.5` | Default job image; can be overridden per job |
| `privileged` | `false` | Not required for socket binding |
| `network_mode` | `host` | Jobs reach GitLab at `localhost:8000` (or your port) |
| `volumes` | socket + cache | Docker builds + dependency caching |
| `pull_policy` | `if-not-present` | Skip re-pulling already cached images |

### Optional: Enable Prometheus metrics

Add to top of `config.toml` if you need runner monitoring:

```toml
listen_address = ":9252"
```

---

## 5. Runner Registration (GitLab 18.x)

> ⚠️ The old `GITLAB_SHARED_RUNNERS_REGISTRATION_TOKEN` method is **removed** in GitLab 16+.  
> Use the new token-based registration flow below.

### Step 1 — Create a runner in GitLab UI

1. Go to: **Admin Area → CI/CD → Runners → New instance runner**
2. Select platform: **Linux**
3. Add tags (e.g., `docker`)
4. Click **Create runner**
5. Copy the generated token (`glrt-xxxx...`)

### Step 2 — Register the runner

```bash
docker exec -it gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "http://<YOUR_GITLAB_VM_IP_OR_DOMAIN>" \
  --token "glrt-<YOUR_TOKEN>" \
  --executor "docker" \
  --docker-image "docker:27.5" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --docker-network-mode "host" \
  --description "docker-runner"
```

This writes to `/etc/gitlab-runner/config.toml` inside the container (persisted via your `./config` volume mount).

### Verify runner is online

- **GitLab Admin Area → CI/CD → Runners** — runner should appear **green (online)**

---

## 6. GitLab CI Pipeline Template

Full pipeline covering build + SSH deploy, with protection rules:

```yaml
stages:
  - build
  - deploy

# ─── BUILD ────────────────────────────────────────────────────────────────────
build:
  stage: build
  image: docker:27.5
  tags:
    - docker
  script:
    - docker build -t $CI_REGISTRY_IMAGE:${CI_COMMIT_SHA:0:8} .
    - echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
    - docker push $CI_REGISTRY_IMAGE:${CI_COMMIT_SHA:0:8}
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"   # validate on MR
    - if: $CI_COMMIT_BRANCH == "main"                    # build on merge

# ─── DEPLOY ───────────────────────────────────────────────────────────────────
deploy:
  stage: deploy
  image: alpine:latest
  tags:
    - docker
  before_script:
    - apk add --no-cache openssh-client
    - chmod 400 "$SSH_PRIVATE_KEY"
  script:
    - ssh -i "$SSH_PRIVATE_KEY" -o StrictHostKeyChecking=no user@your-server \
        "docker pull $CI_REGISTRY_IMAGE:${CI_COMMIT_SHA:0:8} &&
         docker stop myapp || true &&
         docker rm myapp || true &&
         docker run -d --name myapp --restart always $CI_REGISTRY_IMAGE:${CI_COMMIT_SHA:0:8}"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual        # requires clicking ▶ Play in GitLab UI
```

### Required GitLab CI/CD Variables

**Project → Settings → CI/CD → Variables**

| Variable | Type | Description |
|---|---|---|
| `SSH_PRIVATE_KEY` | File | Private key for SSH access to deployment servers |
| `CI_REGISTRY_USER` | Variable | Docker registry username (auto-set by GitLab) |
| `CI_REGISTRY_PASSWORD` | Variable | Docker registry password (auto-set by GitLab) |

> `CI_REGISTRY`, `CI_REGISTRY_IMAGE`, `CI_COMMIT_SHA` are auto-injected by GitLab — no need to set manually.

---

## 7. Deployment Approval Strategy (CE Edition)

> GitLab CE does **not** support Environment Deployment Approvals (Premium feature).  
> The recommended CE alternative is: **Protected branch + MR approval + `when: manual`**

### Layer 1 — Protect the `main` branch

**Project → Settings → Repository → Protected Branches**

```
Branch:           main
Allowed to merge: Developers + Maintainers
Allowed to push:  No one (or Maintainers only)
Require MR:       ✅ enabled
```

### Layer 2 — Require MR Approval (CE: 1 rule)

**Project → Settings → Merge Requests → Approval Rules**

```
Approvals required: 1
Approver:           [your lead / reviewer]
✅ Prevent merge unless approved
✅ Reset approvals on new commits
```

> CE supports **1 approval rule only**. Multiple rules require Premium.

### Layer 3 — Manual deploy gate in pipeline

```yaml
deploy:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual    # human must click ▶ Play even after merge
```

### Full approval flow

```
Developer pushes feature branch
        ↓
MR opened → build job runs (validates) ✅
        ↓
Reviewer reviews code + approves MR ✅
        ↓
Lead merges to main
        ↓
Pipeline triggers on main:
  → build  ✅ (automatic)
  → deploy ⏸ (manual — click ▶ Play in GitLab pipeline UI)
        ↓
Deploy job runs SSH commands on server ✅
```

### CE vs Premium feature comparison

| Feature | CE | Premium |
|---|---|---|
| Protected branches | ✅ | ✅ |
| MR required before merge | ✅ | ✅ |
| 1 MR approval rule | ✅ | ✅ |
| Multiple MR approval rules | ❌ | ✅ |
| Environment deployment approvals | ❌ | ✅ |
| `when: manual` deploy gate | ✅ | ✅ |

**Recommendation for CE:** All 3 layers together (protected branch + MR approval + `when: manual`) provide a solid, cost-free approval workflow.

---

## 8. Healthy Startup Log Reference

When the runner starts correctly you should see:

```
Runtime platform    arch=amd64 os=linux pid=7 revision=ad1797b3 version=18.11.3
Starting multi-runner from /etc/gitlab-runner/config.toml...
Running in system-mode.
Usage logger disabled                builds=0 max_builds=4
Configuration loaded                 builds=0 max_builds=4
listen_address not defined, metrics & debug endpoints disabled
[session_server].listen_address not defined, session endpoints disabled
Initializing executor providers      builds=0 max_builds=4
```

| Line | Meaning | Status |
|---|---|---|
| `version=18.11.3` | Runner version matches GitLab server | ✅ |
| `config.toml` loaded | Config file found and parsed | ✅ |
| `max_builds=4` | `concurrent = 4` applied | ✅ |
| `metrics disabled` | No `listen_address` set — optional | ℹ️ |
| `session endpoints disabled` | Interactive terminal off — not needed | ℹ️ |
| `Initializing executor providers` | Docker executor ready, waiting for jobs | ✅ |

---

## 9. Security Notes

### Issues found in the reference repository (do not replicate)

| Issue | Risk | Fix |
|---|---|---|
| Hardcoded root password in `docker-compose.yml` | Anyone with repo access can log in | Use `.env` file: `${GITLAB_ROOT_PASSWORD}` |
| Runner tokens (`glrt-*`) committed to git | Tokens in permanent git history | Use placeholder `<YOUR_TOKEN>`, set via env var |
| Unpinned `:latest` image tags | Silent breaking changes on pull | Pin to specific version e.g. `v18.11.3` |
| Docker socket binding | Gives CI jobs root-equivalent Docker host access | Acceptable for dedicated CI VMs; document clearly |
| `network_mode: host` Linux-only | Fails silently on macOS/Windows Docker Desktop | Document host OS requirement |

### SSH key best practices

- Store `SSH_PRIVATE_KEY` as a **GitLab CI/CD File variable** (not Variable) — it writes to a temp file automatically
- Use a **dedicated deploy keypair** — not your personal SSH key
- On target servers, add only the **public key** to `~/.ssh/authorized_keys` of a restricted deploy user
- Restrict the deploy user to only the commands needed (via `authorized_keys` `command=` option or `sudoers`)

---

*Generated from GitLab-in-Docker repository analysis — session 2026-05-12*
