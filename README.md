# complete-cicd

Application full-stack Python (backend FastAPI + frontend Flask) servant de **terrain d'étude DevOps complet** : du code local jusqu'au déploiement Kubernetes via GitOps, en passant par CI, releases automatisées, et conteneurs publiés sur GHCR.

Ce repo héberge **le code applicatif et l'outillage**. Un **second repo séparé** (`complete-cicd-gitops`) héberge **l'état désiré du cluster** ; Flux le surveille et le réconcilie en continu.

---

## 1. Architecture en deux repos

```
┌──────────────────────────────────┐         ┌──────────────────────────────────┐
│  complete-cicd  (CE REPO)        │         │  complete-cicd-gitops            │
│                                  │         │                                  │
│  src/backend/   code FastAPI     │         │  clusters/dev/                   │
│  src/frontend/  code Flask       │         │    flux-system/   (auto Flux)    │
│  src/demo/      app de démo      │         │  apps/                           │
│  kubernetes/    manifests + scr. │         │    dev/kustomization.yaml        │
│  .github/       workflows CI     │         │    prod/kustomization.yaml       │
│  scripts/       outils dev/CI    │         │                                  │
│                                  │         │  ← Flux pull en boucle           │
│  → écrit par les devs            │         │  ← écrit par la CI (auto)        │
│  → CI build & push images        │         │  ← humain merge PR pour prod     │
└──────────────────────────────────┘         └──────────────────────────────────┘
            │                                            ▲
            │ build & push                               │ pull
            ▼                                            │
        ghcr.io/thomaspretat/study-app-{api,web}:<tag> ──┘
```

**Pourquoi cette séparation ?** C'est le cœur du GitOps : le code source et l'état du cluster ne vivent pas au même rythme et n'ont pas les mêmes lecteurs. Le repo gitops est un journal Git de ce qui tourne en cluster — `git revert` = rollback en quelques secondes.

---

## 2. Stack et outillage

| Outil | Rôle | Géré par |
|---|---|---|
| [mise](https://mise.jdx.dev) | Gestion des versions de tous les outils + tasks | `mise.toml` |
| [uv](https://docs.astral.sh/uv/) | Gestion Python (venv, deps, lockfile) | `pyproject.toml` + `uv.lock` |
| [ruff](https://docs.astral.sh/ruff/) | Lint + format Python | `pre-commit` + CI |
| [pre-commit](https://pre-commit.com) + [commitizen](https://commitizen-tools.github.io/commitizen/) | Hooks Git, validation des messages de commit (Conventional Commits) | `.pre-commit-config.yaml` |
| [k3d](https://k3d.io) | Cluster Kubernetes local (k3s dans Docker) | `kubernetes/k3d-config.yaml` |
| [kubectl](https://kubernetes.io/docs/reference/kubectl/) + [Kustomize](https://kustomize.io) | Déploiement et overlays par environnement | `kubernetes/manifests/` |
| [Flux](https://fluxcd.io) | GitOps controller (pull du repo gitops, applique au cluster) | `kubernetes/setup_gitops` |
| [Trivy](https://trivy.dev) | Scan de vulnérabilités sur les images Docker | CI |
| [release-please](https://github.com/googleapis/release-please) | Bump de version + changelog + tags depuis les commits | `release-please-config.json` |
| **Devcontainer** | Environnement reproductible (VS Code / GitHub Codespaces) | `.devcontainer/` |

`mise.toml` est le **point d'entrée unique** : il déclare toutes les versions d'outils, des variables d'env, et des tasks (`mise run <task>`). Quand tu `cd` dans le repo, mise installe automatiquement tout ce qu'il faut.

---

## 3. Structure du repo

```
complete-cicd/
├── .devcontainer/             Image Ubuntu + mise pour Codespaces / Dev Containers
├── .github/workflows/         CI GitHub Actions (cf. §5)
│   ├── backend-tests.yaml     Tests + lint + Trivy sur PR backend
│   ├── frontend-tests.yaml    Idem frontend
│   ├── e2e-tests.yaml         Tests E2E (cluster k3d éphémère)
│   ├── release-please.yaml    Crée des PR de release sur main
│   ├── docker-build-push.yaml Push images sur GHCR quand tag
│   └── update-gitops.yaml     Met à jour le repo gitops
├── src/
│   ├── backend/   FastAPI, port 22112, /sessions /stats /health
│   ├── frontend/  Flask, port 22111, templates Jinja
│   └── demo/      Petite démo annexe
├── kubernetes/
│   ├── k3d-config.yaml        1 server + 1 agent, traefik désactivé
│   ├── manifests/
│   │   ├── base/              Manifests génériques (Deployment+Service)
│   │   └── dev/               Overlay dev (namespace, prefix "dev-", env)
│   ├── setup_cluster_local    Cluster + build + apply manifests
│   ├── setup_cluster_minimal  Cluster vide (sans deploy)
│   ├── setup_cluster_gitops   Cluster + Flux + bootstrap GitOps
│   ├── setup_gitops           Bootstrap Flux seul
│   └── e2e_test.py            Tests E2E pilotant un cluster k3d
├── scripts/
│   ├── setup                  Installe mise (postCreateCommand)
│   ├── setup_project          Installe commitizen + hooks Git (mise enter hook)
│   ├── setup_deploy_key       Génère SSH key + l'ajoute en deploy key GitHub
│   └── update_kustomize_tag   yq sur kustomization.yaml (utilisé par CI)
├── mise.toml                  Tools + tasks + env
├── release-please-config.json Config release-please (2 packages: backend, frontend)
└── .pre-commit-config.yaml    Hooks ruff + commitizen
```

---

## 4. Onboarding et dev local

### 4.1 Le boot d'un environnement

```
ouverture du repo (Codespace / Devcontainer / clone local)
        │
        ├── (Devcontainer/Codespace)  postCreateCommand → scripts/setup
        │                                                   │
        │                                                   ▼
        │                                        mise trust + mise install
        │                                        (installe tous les tools)
        │
        └── (mise) hook "enter" → scripts/setup_project
                                              │
                                              ▼
                                  pipx install commitizen
                                  pre-commit install (hooks ruff + commit-msg)
```

Au final tu as : Python 3.13, uv, kubectl, k3d, k9s, flux2, gh, trivy, ruff, pre-commit, commitizen — tout pinné par mise.

### 4.2 Tasks mise disponibles

```bash
mise run k8s-setup-local     # cluster k3d + build images locales + apply manifests
mise run k8s-setup-minimal   # juste un cluster k3d vide
mise run k8s-setup-gitops    # cluster + Flux + bootstrap GitOps
mise run e2e-test            # tests E2E sur cluster éphémère
mise run setup-keys          # génère et upload la deploy key vers le repo gitops
```

### 4.3 Variables d'environnement

Dans `mise.toml [env]` :
```toml
APP_REPO = 'complete-cicd'
GITOPS_REPO = 'complete-cicd-gitops'
```

Les scripts résolvent automatiquement `WORKSPACE_ID` et `GITUSER` selon l'environnement (DevPod → Codespaces → local) — pas besoin de configurer quoi que ce soit dans un Codespace standard.

---

## 5. CI/CD : qui appelle quoi, quand

Cinq workflows GitHub Actions, déclenchés à des moments différents.

### 5.1 Sur Pull Request — qualité du code

```
PR ouverte
   │
   ├─ backend-tests.yaml       (si src/backend/** modifié)
   │     ├─ uv sync
   │     ├─ ruff check + format
   │     ├─ pytest --cov-fail-under=80
   │     ├─ docker build
   │     ├─ trivy scan (CRITICAL/HIGH)
   │     └─ trigger e2e-tests.yaml
   │
   ├─ frontend-tests.yaml      (si src/frontend/** modifié)
   │     └─ même chose, mais ne déclenche pas E2E
   │
   └─ e2e-tests.yaml           (workflow_call uniquement)
         ├─ install k3d + kubectl + uv
         └─ uv run e2e_test.py
            (build images, crée cluster, applique manifests, hit endpoints)
```

L'utilisation de `dorny/paths-filter` évite de rejouer toute la CI si seul un fichier de doc change.

### 5.2 Sur push main — release automatique

```
push sur main
   │
   └─ release-please.yaml
         │
         └─ Lit les commits Conventional Commits depuis la dernière release.
            Selon les types (feat:, fix:, chore:), ouvre/maintient une PR
            "release: backend X.Y.Z" et/ou "release: frontend X.Y.Z" qui :
              - bump la version dans pyproject.toml et uv.lock
              - met à jour CHANGELOG.md
              - quand mergée, crée un tag "backend-vX.Y.Z" / "frontend-vX.Y.Z"
                et la GitHub Release associée
```

Configuré pour 2 packages indépendants (`src/backend` et `src/frontend`) via `release-please-config.json` + manifest.

### 5.3 Sur tag — build, push, propagation GitOps

```
tag "backend-v1.2.3" ou "frontend-v0.5.1" pushé
   │
   └─ docker-build-push.yaml
         │
         ├─ build-and-push-backend   (si tag commence par "backend")
         │     └─ docker build + push ghcr.io/.../study-app-api:<tag> + :latest
         │
         ├─ build-and-push-frontend  (si tag commence par "frontend")
         │     └─ docker build + push ghcr.io/.../study-app-web:<tag> + :latest
         │
         └─ trigger_gitops → update-gitops.yaml
               │
               ├─ checkout repo gitops (via SSH deploy key)
               ├─ scripts/update_kustomize_tag → maj apps/dev/kustomization.yaml
               ├─ commit + push direct sur main du repo gitops          ← auto-deploy DEV
               ├─ scripts/update_kustomize_tag → maj apps/prod/kustomization.yaml
               └─ ouvre une PR sur le repo gitops                        ← review humaine PROD
                  (titre "Update prod image tag to <tag>")
```

**Stratégie clé** : dev = push direct (vitesse), prod = PR (sécurité). Un humain merge la PR prod quand il est prêt.

### 5.4 Côté cluster — Flux

Une fois `mise run k8s-setup-gitops` exécuté une fois sur un cluster :

```
Flux (dans le namespace flux-system)
   │
   ├─ source-controller : clone et watch le repo gitops (poll ~1 min)
   │
   └─ kustomize-controller :
         détecte un changement → kubectl apply -k clusters/dev → réconciliation
```

Donc dès que la CI commit dans le repo gitops, Flux propage le nouveau tag d'image au cluster en moins d'une minute, sans intervention manuelle.

---

## 6. Deux repos, deux flux Kubernetes

Le `kubernetes/` de **CE repo** sert à **3 choses**, dont aucune n'est l'état désiré du cluster :

1. **Dev local** ([setup_cluster_local](kubernetes/setup_cluster_local)) — build images locales `:dev`, apply direct, pas de Flux.
2. **Source des manifests "base"** — le code de référence des Deployments/Services. Le repo gitops les référence ou en hérite.
3. **Bootstrap GitOps** ([setup_cluster_gitops](kubernetes/setup_cluster_gitops)) — installe Flux *une fois*, puis Flux prend la main.

Le `kubernetes/` du **repo gitops** est l'**état désiré permanent**. Il n'est jamais lancé à la main, seul Flux le consomme.

---

## 7. Réseau dans k3d (k3s dans Docker)

Le cluster local crée 2 containers Docker (server + agent) qui sont les nodes Kubernetes. Trois réseaux superposés :

| Réseau | Plage | Rôle |
|---|---|---|
| Docker bridge | `172.19.0.0/16` | IPs réelles des containers nodes |
| Pods (CNI flannel) | `10.42.0.0/16` | IPs des pods |
| Services (virtuel) | `10.43.0.0/16` | ClusterIPs (n'existent qu'en règles iptables) |

Trois acteurs routent le trafic :
- **CoreDNS** : résolution de noms de Service (interne)
- **kube-proxy** : règles iptables sur chaque node, route ClusterIP → pod
- **klipper-lb** : pods `svclb-*` qui bind les ports des Services `LoadBalancer` sur les IPs des nodes (donne une "EXTERNAL-IP" en local)

Traefik est désactivé via `--disable=traefik` dans `k3d-config.yaml` car on n'a pas besoin de routage HTTP par hostname/path : 2 services exposés directement par leur port.

---

## 8. Quickstart

### En local (ou Codespace)

```bash
# 1. Tout est prêt grâce au devcontainer + mise. Sinon en local :
mise install

# 2. Lancer l'app en local sur un cluster k3d
mise run k8s-setup-local
# → URLs frontend / backend affichées en sortie

# 3. Tester end-to-end
mise run e2e-test
```

### Mettre en place le pipeline GitOps (à faire une fois)

```bash
# 1. Créer le repo gitops sur GitHub
gh repo create thomaspretat/complete-cicd-gitops --private

# 2. Y déposer une structure minimale :
#      clusters/dev/   apps/dev/kustomization.yaml   apps/prod/kustomization.yaml

# 3. Créer la deploy key et l'uploader
mise run setup-keys

# 4. Bootstrap Flux sur le cluster
mise run k8s-setup-gitops

# 5. Côté GitHub repo settings, créer 2 secrets :
#    - GITOPS_DEPLOY_KEY  (la clé privée /workspaces/.../dev-keys/study_app_gitops_deploy_key)
#    - DEVOPS_STUDY_APP   (un PAT classique avec scope "repo" sur le repo gitops)
```

À partir de là, tout commit conforme à Conventional Commits sur `main` déclenche release-please → tag → build → push GHCR → update gitops → Flux → cluster.

---

## 9. Le flux complet bout en bout

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │  1. dev push  feat: ...  (commit conventionnel)                      │
   └──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  2. CI PR : ruff + pytest + trivy + e2e-test                         │
   └──────────────────────────────────────────────────────────────────────┘
                                  │  (merge)
                                  ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  3. release-please ouvre/maintient une PR de release                 │
   │     → mergée → tag "backend-vX.Y.Z" créé                             │
   └──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  4. docker-build-push : build + push ghcr.io/.../study-app-api:tag   │
   └──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  5. update-gitops :                                                  │
   │     - apps/dev/kustomization.yaml  → push direct main                │
   │     - apps/prod/kustomization.yaml → PR à valider                    │
   └──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  6. Flux dans le cluster détecte → kubectl apply → rollout pod       │
   └──────────────────────────────────────────────────────────────────────┘
```

Aucune main humaine entre l'étape 3 et le déploiement en dev. Pour la prod, une seule action manuelle : merger la PR.

---

## 10. Commandes utiles au quotidien

```bash
# Cluster
k3d cluster list
k3d cluster delete study-app-cluster
k9s -A                                    # TUI pour explorer le cluster

# Kubernetes
kubectl get pods -n study-app
kubectl get svc -A
kubectl logs -n study-app -l component=backend -f

# Flux (si bootstrappé)
flux get all
flux reconcile source git flux-system     # forcer un pull immédiat

# Dev Python
uv sync --dev                              # depuis src/backend ou src/frontend
uv run pytest

# Releases manuelles (cas de figure rare)
git tag backend-v1.2.3 && git push --tags  # déclenche docker-build-push directement
```
