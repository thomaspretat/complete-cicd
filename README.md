# complete-cicd

A full-stack Python application (FastAPI backend + Flask frontend) used as a **complete DevOps playground**: from local development to Kubernetes deployment via GitOps, with CI, automated releases, and container images published to GHCR.

This repo holds **the application code and tooling**. A **separate repo** (`complete-cicd-gitops`) holds **the desired cluster state**; Flux watches it and reconciles continuously.

---

## 1. Two-repo architecture

```
┌──────────────────────────────────┐         ┌──────────────────────────────────┐
│  complete-cicd  (THIS REPO)      │         │  complete-cicd-gitops            │
│                                  │         │                                  │
│  src/backend/   FastAPI code     │         │  clusters/dev/                   │
│  src/frontend/  Flask code       │         │    flux-system/   (Flux auto)    │
│  src/demo/      demo app         │         │  apps/                           │
│  kubernetes/    manifests + scr. │         │    dev/kustomization.yaml        │
│  .github/       CI workflows     │         │    prod/kustomization.yaml       │
│  scripts/       dev/CI tools     │         │                                  │
│                                  │         │  ← Flux pulls in a loop          │
│  → written by developers         │         │  ← written by CI (auto)          │
│  → CI builds & pushes images     │         │  ← humans merge PR for prod      │
└──────────────────────────────────┘         └──────────────────────────────────┘
            │                                            ▲
            │ build & push                               │ pull
            ▼                                            │
        ghcr.io/thomaspretat/study-app-{api,web}:<tag> ──┘
```

**Why this split?** That's the heart of GitOps: source code and cluster state don't move at the same pace and don't have the same readers. The gitops repo is a Git journal of what's running in the cluster — `git revert` = rollback in seconds.

---

## 2. Stack and tooling

| Tool | Role | Managed by |
|---|---|---|
| [mise](https://mise.jdx.dev) | Version management for all tools + tasks | `mise.toml` |
| [uv](https://docs.astral.sh/uv/) | Python management (venv, deps, lockfile) | `pyproject.toml` + `uv.lock` |
| [ruff](https://docs.astral.sh/ruff/) | Python lint + format | `pre-commit` + CI |
| [pre-commit](https://pre-commit.com) + [commitizen](https://commitizen-tools.github.io/commitizen/) | Git hooks, commit message validation (Conventional Commits) | `.pre-commit-config.yaml` |
| [k3d](https://k3d.io) | Local Kubernetes cluster (k3s in Docker) | `kubernetes/k3d-config.yaml` |
| [kubectl](https://kubernetes.io/docs/reference/kubectl/) + [Kustomize](https://kustomize.io) | Deployment and per-environment overlays | `kubernetes/manifests/` |
| [Flux](https://fluxcd.io) | GitOps controller (pulls the gitops repo, applies to cluster) | `kubernetes/setup_gitops` |
| [Trivy](https://trivy.dev) | Vulnerability scanning on Docker images | CI |
| [release-please](https://github.com/googleapis/release-please) | Version bump + changelog + tags from commits | `release-please-config.json` |
| **Devcontainer** | Reproducible environment (VS Code / GitHub Codespaces) | `.devcontainer/` |

`mise.toml` is the **single entry point**: it declares all tool versions, environment variables, and tasks (`mise run <task>`). When you `cd` into the repo, mise installs everything you need automatically.

---

## 3. Repo layout

```
complete-cicd/
├── .devcontainer/             Ubuntu image + mise for Codespaces / Dev Containers
├── .github/workflows/         GitHub Actions CI (see §5)
│   ├── backend-tests.yaml     Tests + lint + Trivy on backend PR
│   ├── frontend-tests.yaml    Same for frontend
│   ├── e2e-tests.yaml         E2E tests (ephemeral k3d cluster)
│   ├── release-please.yaml    Opens release PRs on main
│   ├── docker-build-push.yaml Pushes images to GHCR on tag
│   └── update-gitops.yaml     Updates the gitops repo
├── src/
│   ├── backend/   FastAPI, port 22112, /sessions /stats /health
│   ├── frontend/  Flask, port 22111, Jinja templates
│   └── demo/      Small side demo
├── kubernetes/
│   ├── k3d-config.yaml        1 server + 1 agent, traefik disabled
│   ├── manifests/
│   │   ├── base/              Generic manifests (Deployment+Service)
│   │   └── dev/               Dev overlay (namespace, "dev-" prefix, env)
│   ├── setup_cluster_local    Cluster + build + apply manifests
│   ├── setup_cluster_minimal  Empty cluster (no deploy)
│   ├── setup_cluster_gitops   Cluster + Flux + GitOps bootstrap
│   ├── setup_gitops           Flux bootstrap only
│   └── e2e_test.py            E2E tests driving a k3d cluster
├── scripts/
│   ├── setup                  Installs mise (postCreateCommand)
│   ├── setup_project          Installs commitizen + Git hooks (mise enter hook)
│   ├── setup_deploy_key       Generates an SSH key + adds it as a GitHub deploy key
│   └── update_kustomize_tag   yq on kustomization.yaml (used by CI)
├── mise.toml                  Tools + tasks + env
├── release-please-config.json release-please config (2 packages: backend, frontend)
└── .pre-commit-config.yaml    ruff + commitizen hooks
```

---

## 4. Onboarding and local dev

### 4.1 Environment boot

```
repo opened (Codespace / Devcontainer / local clone)
        │
        ├── (Devcontainer/Codespace)  postCreateCommand → scripts/setup
        │                                                   │
        │                                                   ▼
        │                                        mise trust + mise install
        │                                        (installs every tool)
        │
        └── (mise) "enter" hook → scripts/setup_project
                                              │
                                              ▼
                                  pipx install commitizen
                                  pre-commit install (ruff + commit-msg hooks)
```

End result: Python 3.13, uv, kubectl, k3d, k9s, flux2, gh, trivy, ruff, pre-commit, commitizen — all pinned by mise.

### 4.2 Available mise tasks

```bash
mise run k8s-setup-local     # k3d cluster + build local images + apply manifests
mise run k8s-setup-minimal   # empty k3d cluster only
mise run k8s-setup-gitops    # cluster + Flux + GitOps bootstrap
mise run e2e-test            # E2E tests on an ephemeral cluster
mise run setup-keys          # generate and upload deploy key to the gitops repo
```

### 4.3 Environment variables

In `mise.toml [env]`:
```toml
APP_REPO = 'complete-cicd'
GITOPS_REPO = 'complete-cicd-gitops'
```

Scripts automatically resolve `WORKSPACE_ID` and `GITUSER` based on the environment (DevPod → Codespaces → local) — no extra configuration needed in a standard Codespace.

---

## 5. CI/CD: who calls what, when

Five GitHub Actions workflows, triggered at different moments.

### 5.1 On Pull Request — code quality

```
PR opened
   │
   ├─ backend-tests.yaml       (if src/backend/** modified)
   │     ├─ uv sync
   │     ├─ ruff check + format
   │     ├─ pytest --cov-fail-under=80
   │     ├─ docker build
   │     ├─ trivy scan (CRITICAL/HIGH)
   │     └─ trigger e2e-tests.yaml
   │
   ├─ frontend-tests.yaml      (if src/frontend/** modified)
   │     └─ same flow, but does not trigger E2E
   │
   └─ e2e-tests.yaml           (workflow_call only)
         ├─ install k3d + kubectl + uv
         └─ uv run e2e_test.py
            (build images, create cluster, apply manifests, hit endpoints)
```

Using `dorny/paths-filter` avoids re-running the whole CI when only a doc file changes.

### 5.2 On push to main — automated release

```
push to main
   │
   └─ release-please.yaml
         │
         └─ Reads Conventional Commits since the last release.
            Based on types (feat:, fix:, chore:), opens/maintains a PR
            "release: backend X.Y.Z" and/or "release: frontend X.Y.Z" that:
              - bumps the version in pyproject.toml and uv.lock
              - updates CHANGELOG.md
              - when merged, creates a "backend-vX.Y.Z" / "frontend-vX.Y.Z" tag
                and the corresponding GitHub Release
```

Configured for 2 independent packages (`src/backend` and `src/frontend`) via `release-please-config.json` + manifest.

### 5.3 On tag — build, push, GitOps propagation

```
tag "backend-v1.2.3" or "frontend-v0.5.1" pushed
   │
   └─ docker-build-push.yaml
         │
         ├─ build-and-push-backend   (if tag starts with "backend")
         │     └─ docker build + push ghcr.io/.../study-app-api:<tag> + :latest
         │
         ├─ build-and-push-frontend  (if tag starts with "frontend")
         │     └─ docker build + push ghcr.io/.../study-app-web:<tag> + :latest
         │
         └─ trigger_gitops → update-gitops.yaml
               │
               ├─ checkout gitops repo (via SSH deploy key)
               ├─ scripts/update_kustomize_tag → update apps/dev/kustomization.yaml
               ├─ commit + push directly to main of the gitops repo         ← auto-deploy DEV
               ├─ scripts/update_kustomize_tag → update apps/prod/kustomization.yaml
               └─ open a PR on the gitops repo                              ← human review for PROD
                  (title "Update prod image tag to <tag>")
```

**Key strategy**: dev = direct push (speed), prod = PR (safety). A human merges the prod PR when ready.

### 5.4 In the cluster — Flux

Once `mise run k8s-setup-gitops` has been run on a cluster:

```
Flux (in the flux-system namespace)
   │
   ├─ source-controller : clones and watches the gitops repo (poll ~1 min)
   │
   └─ kustomize-controller :
         detects a change → kubectl apply -k clusters/dev → reconciliation
```

So as soon as the CI commits to the gitops repo, Flux propagates the new image tag to the cluster in under a minute, with no manual intervention.

---

## 6. Two repos, two Kubernetes flows

The `kubernetes/` folder in **THIS repo** serves **3 purposes**, none of which is the cluster's desired state:

1. **Local dev** ([setup_cluster_local](kubernetes/setup_cluster_local)) — build local `:dev` images, apply directly, no Flux.
2. **Source of "base" manifests** — the reference Deployments/Services. The gitops repo references or inherits from them.
3. **GitOps bootstrap** ([setup_cluster_gitops](kubernetes/setup_cluster_gitops)) — installs Flux *once*, then Flux takes over.

The `kubernetes/` folder of the **gitops repo** is the **permanent desired state**. It is never run by hand; only Flux consumes it.

---

## 7. Networking in k3d (k3s in Docker)

The local cluster spawns 2 Docker containers (server + agent) which act as the Kubernetes nodes. Three overlapping networks:

| Network | Range | Role |
|---|---|---|
| Docker bridge | `172.19.0.0/16` | Real IPs of the node containers |
| Pods (CNI flannel) | `10.42.0.0/16` | Pod IPs |
| Services (virtual) | `10.43.0.0/16` | ClusterIPs (only exist as iptables rules) |

Three actors route traffic:
- **CoreDNS**: Service name resolution (internal)
- **kube-proxy**: iptables rules on each node, routes ClusterIP → pod
- **klipper-lb**: `svclb-*` pods that bind `LoadBalancer` Service ports on node IPs (gives an "EXTERNAL-IP" locally)

Traefik is disabled via `--disable=traefik` in `k3d-config.yaml` since we don't need HTTP routing by hostname/path: 2 services exposed directly by their port.

---

## 8. Quickstart

### Locally (or in a Codespace)

```bash
# 1. Everything is ready thanks to the devcontainer + mise. Otherwise locally:
mise install

# 2. Run the app on a local k3d cluster
mise run k8s-setup-local
# → frontend / backend URLs printed at the end

# 3. Run end-to-end tests
mise run e2e-test
```

### Setting up the GitOps pipeline (one-time)

```bash
# 1. Create the gitops repo on GitHub
gh repo create thomaspretat/complete-cicd-gitops --private

# 2. Seed it with a minimal layout:
#      clusters/dev/   apps/dev/kustomization.yaml   apps/prod/kustomization.yaml

# 3. Generate the deploy key and upload it
mise run setup-keys

# 4. Bootstrap Flux on the cluster
mise run k8s-setup-gitops

# 5. In GitHub repo settings, create 2 secrets:
#    - GITOPS_DEPLOY_KEY  (the private key /workspaces/.../dev-keys/study_app_gitops_deploy_key)
#    - DEVOPS_STUDY_APP   (a classic PAT with "repo" scope on the gitops repo)
```

From then on, any Conventional-Commit-compliant commit on `main` triggers release-please → tag → build → push GHCR → update gitops → Flux → cluster.

---

## 9. End-to-end flow

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │  1. dev pushes  feat: ...  (conventional commit)                     │
   └──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  2. PR CI: ruff + pytest + trivy + e2e-test                          │
   └──────────────────────────────────────────────────────────────────────┘
                                  │  (merge)
                                  ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  3. release-please opens/maintains a release PR                      │
   │     → merged → "backend-vX.Y.Z" tag created                          │
   └──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  4. docker-build-push: build + push ghcr.io/.../study-app-api:tag    │
   └──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  5. update-gitops:                                                   │
   │     - apps/dev/kustomization.yaml  → direct push to main             │
   │     - apps/prod/kustomization.yaml → PR to review                    │
   └──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  6. Flux in the cluster detects the change → kubectl apply → rollout │
   └──────────────────────────────────────────────────────────────────────┘
```

No human in the loop between step 3 and the dev deployment. For prod, a single manual action: merging the PR.

---

## 10. Useful day-to-day commands

```bash
# Cluster
k3d cluster list
k3d cluster delete study-app-cluster
k9s -A                                    # TUI to explore the cluster

# Kubernetes
kubectl get pods -n study-app
kubectl get svc -A
kubectl logs -n study-app -l component=backend -f

# Flux (once bootstrapped)
flux get all
flux reconcile source git flux-system     # force an immediate pull

# Python dev
uv sync --dev                              # from src/backend or src/frontend
uv run pytest

# Manual releases (rare case)
git tag backend-v1.2.3 && git push --tags  # triggers docker-build-push directly
```
