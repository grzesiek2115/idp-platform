# Internal Developer Platform (IDP)

A production-grade Internal Developer Platform built on Kubernetes, demonstrating GitOps-first Platform Engineering using [Backstage](https://backstage.io), [ArgoCD](https://argo-cd.readthedocs.io), and [Kind](https://kind.sigs.k8s.io).

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Local Machine                               │
│                                                                      │
│  ┌─────────────────────────────────────────────┐                    │
│  │           IDP Cluster  (:8080/:8443)         │                    │
│  │                                              │                    │
│  │  ┌────────────┐    ┌──────────────────────┐ │                    │
│  │  │  Backstage  │    │   ArgoCD  (hub)      │ │                    │
│  │  │  :7007      │    │   manages all others │ │                    │
│  │  └────────────┘    └──────────┬───────────┘ │                    │
│  │  ┌────────────┐               │              │                    │
│  │  │ cert-manager│              │ cluster API  │                    │
│  │  └────────────┘              sync            │                    │
│  │  ┌────────────┐               │              │                    │
│  │  │    ESO     │               │              │                    │
│  │  └────────────┘               │              │                    │
│  └───────────────────────────────┼──────────────┘                   │
│                                  │                                   │
│         ┌────────────────────────┼──────────────────┐               │
│         │                        │                  │               │
│  ┌──────▼──────┐     ┌───────────▼────┐  ┌─────────▼──────┐       │
│  │  DEV Cluster │     │ STAGE Cluster  │  │ PROD Cluster   │       │
│  │  :9080/:9443 │     │ :9180/:9543    │  │ :9280/:9643    │       │
│  │              │     │                │  │                │       │
│  │  ingress-nginx│    │  ingress-nginx  │  │  ingress-nginx │       │
│  │  ESO         │     │  ESO           │  │  ESO           │       │
│  └──────────────┘     └────────────────┘  └────────────────┘       │
└──────────────────────────────────────────────────────────────────────┘
```

### GitOps Flow

```
Git push → ArgoCD detects change → reconciles cluster state
                                         │
                              clusters/idp/       → IDP platform components
                              clusters/workloads/ → ApplicationSets → DEV/STAGE/PROD
```

### Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| GitOps engine | ArgoCD | Hub-spoke multi-cluster, ApplicationSets, mature ecosystem |
| Bootstrap pattern | App of Apps | One manual apply, rest is self-managed |
| Workload management | ApplicationSets + cluster generator | DRY — one manifest drives all workload clusters |
| Backstage deployment | ArgoCD multi-source | Helm chart + values from same repo, no manual Helm commands |
| TLS | cert-manager + self-signed CA | Realistic local setup, same pattern as production |
| Secrets | External Secrets Operator | Decouples secret storage from manifests; swap backend to Vault/AWS SSM with no app change |

## Prerequisites

```bash
# Required tools
brew install kind kubectl helm argocd

# Verify
kind version      # >= 0.24
kubectl version --client
helm version      # >= 3.14
argocd version --client
```

## Quick Start

### 1. Push this repo to GitHub first

ArgoCD pulls from git — it needs a remote URL.

```bash
git init && git add . && git commit -m "feat: initial IDP scaffold"
gh repo create idp-project --private --push --source .
```

### 2. Create Kind clusters

```bash
make clusters-up
```

This creates 4 clusters with proper ingress port mappings:
- `idp-k8s-cluster` → ports 8080/8443
- `dev-k8s-cluster` → ports 9080/9443
- `stage-k8s-cluster` → ports 9180/9543
- `prod-k8s-cluster` → ports 9280/9643

### 3. Bootstrap the platform

```bash
make bootstrap           # install ArgoCD via Kustomize + apply argocd-app.yaml
make register-clusters   # register DEV/STAGE/PROD as ArgoCD spoke clusters
```

`make bootstrap` — no Helm, no scripts, three kubectl commands:
1. `kubectl apply -k argocd/` (first pass — creates CRDs + Deployments; AppProject/Application CRs may fail)
2. `kubectl wait --for=condition=established crd/applications.argoproj.io` — wait for CRDs
3. `kubectl apply -k argocd/` (second pass — all resources succeed)
4. `kubectl apply -f argocd-app.yaml` — ArgoCD now self-manages `argocd/` from git

From this point everything in `argocd/` is reconciled by ArgoCD:
- AppProjects (`argocd/projects/`)
- App of Apps for IDP cluster (`argocd/apps/idp-platform.yaml` → `clusters/idp/`)
- App of Apps for workload clusters (`argocd/apps/workload-platform.yaml` → `clusters/workloads/`)

`make register-clusters` — the only bash script remaining; registers spoke clusters with `environment=workload` label so ApplicationSets pick them up automatically.

### 4. Add /etc/hosts entries

```bash
echo "127.0.0.1 argocd.idp.local backstage.idp.local" | sudo tee -a /etc/hosts
```

### 5. Create secrets

Before Backstage can start, create its secret:

```bash
kubectl --context kind-idp-k8s-cluster create secret generic backstage-secrets \
  --namespace backstage \
  --from-literal=GITHUB_TOKEN=<your-pat> \
  --from-literal=AUTH_GITHUB_CLIENT_ID=<oauth-app-client-id> \
  --from-literal=AUTH_GITHUB_CLIENT_SECRET=<oauth-app-client-secret> \
  --from-literal=ARGOCD_TOKEN=<argocd-api-token>

kubectl --context kind-idp-k8s-cluster create secret generic backstage-postgresql \
  --namespace backstage \
  --from-literal=password=<strong-password>
```

> In production these secrets are managed by External Secrets Operator pulling from AWS SSM or HashiCorp Vault — no manual `kubectl create secret`.

### 6. Access UIs

```bash
make argocd-ui    # https://localhost:8888  (admin / see below)
make backstage-ui # http://localhost:7007

make argocd-password  # prints initial admin password
```

## Repository Structure

```
idp-project/
├── Makefile                          # Day-1 and day-2 operations
├── register-clusters.sh              # Only bash script: register DEV/STAGE/PROD with ArgoCD hub
├── argocd-app.yaml                   # Root Application — ArgoCD self-manages from here
├── kind-envs/                        # Kind cluster configs (4 clusters)
│
├── argocd/                           # Everything ArgoCD manages declaratively (Kustomize)
│   ├── kustomization.yaml            #   Installs ArgoCD + applies all resources below
│   ├── resources/
│   │   ├── namespace.yaml            #   argocd namespace
│   │   ├── argocd-cm.yaml            #   ArgoCD config patch (URL, timeouts)
│   │   ├── argocd-rbac-cm.yaml       #   RBAC policy patch
│   │   ├── argocd-server-patch.yaml  #   --insecure flag patch
│   │   └── argocd-ingress.yaml       #   Ingress → argocd.idp.local
│   ├── projects/
│   │   ├── platform.yaml             #   AppProject: platform infrastructure
│   │   └── workloads.yaml            #   AppProject: dev/stage/prod workloads
│   └── apps/
│       ├── idp-platform.yaml         #   App of Apps → clusters/idp/
│       └── workload-platform.yaml    #   App of Apps → clusters/workloads/
│
├── clusters/                         # What runs where (ArgoCD Application manifests)
│   ├── idp/                          #   IDP cluster — platform components
│   │   ├── cert-manager.yaml         #     sync-wave 1
│   │   ├── cert-manager-issuers.yaml #     sync-wave 2
│   │   ├── ingress-nginx.yaml        #     sync-wave 2
│   │   ├── external-secrets.yaml     #     sync-wave 2
│   │   └── backstage.yaml            #     sync-wave 4 (multi-source)
│   └── workloads/                    #   All workload clusters via ApplicationSets
│       ├── ingress-nginx-appset.yaml #     cluster generator → DEV+STAGE+PROD
│       └── external-secrets-appset.yaml
│
├── platform/                         # How components are configured (Helm values)
│   ├── cert-manager/
│   │   └── issuers/                  #   Self-signed CA + ClusterIssuers
│   └── backstage/
│       ├── values.yaml               #   Helm values (resources, ingress, postgresql)
│       └── app-config.yaml           #   Backstage app-config as ConfigMap
│
├── catalog/                          # Backstage Software Catalog entities
│   ├── all.yaml                      #   Root Location — entry point for catalog
│   ├── domain.yaml                   #   Platform domain
│   ├── systems/idp-platform.yaml     #   IDP system definition
│   ├── components/                   #   Component definitions (backstage, argocd)
│   └── groups/platform-team.yaml     #   Team definition
│
└── scaffold-templates/               # Backstage Software Templates
    └── new-service/
        ├── template.yaml             #   Template spec with parameters + steps
        └── skeleton/                 #   Generated file templates
            ├── catalog-info.yaml
            └── deploy/               #   Kubernetes manifests (Deployment/Service/Ingress)
```

## Platform Components

### ArgoCD (hub)
- Installed on IDP cluster, manages all other clusters
- Two AppProjects: `platform` (infra) and `workloads` (apps)
- Workload clusters registered with `environment=workload` label → picked up automatically by ApplicationSets
- UI: https://argocd.idp.local

### Backstage
- Developer portal: software catalog, scaffolder, TechDocs
- Deployed via ArgoCD multi-source Application (Helm chart from official registry + values from this repo)
- Plugins: Kubernetes, ArgoCD, GitHub integration
- UI: http://backstage.idp.local

### cert-manager
- Manages TLS certificates across the cluster
- Self-signed CA (`idp-ca-issuer`) for local development
- In production: swap to Let's Encrypt ACME or Vault PKI issuer

### External Secrets Operator (ESO)
- Decouples secret storage from Kubernetes manifests
- Installed on all clusters
- In production: configure `ClusterSecretStore` pointing to AWS SSM Parameter Store or HashiCorp Vault

### Ingress NGINX
- Installed on all clusters via ApplicationSet (one manifest, four clusters)
- Uses `hostPort` on Kind control-plane nodes with `ingress-ready=true` label

## Backstage Software Template: New Microservice

The `scaffold-templates/new-service/template.yaml` template provisions a complete service in one click:

1. Collects: service name, description, owning team, target system, port, replicas, language
2. Generates files from skeleton (Deployment, Service, Ingress, catalog-info.yaml)
3. Creates a private GitHub repository
4. Creates an ArgoCD Application targeting the DEV cluster
5. Registers the service in the Backstage catalog

Developer journey: fill a form in Backstage → fully deployed, catalogued microservice in DEV.

## Day-2 Operations

```bash
# Watch ArgoCD sync all apps
argocd app list

# Force sync a specific app
argocd app sync backstage

# Add a new platform component
# → create clusters/idp/<component>.yaml
# → git push → ArgoCD auto-syncs

# Add a new workload to all clusters
# → update clusters/workloads/<component>-appset.yaml
# → git push → ApplicationSet generates apps for all registered clusters
```

## Production Readiness Notes

This project is a local demonstration. Production differences:

| Area | Demo | Production |
|---|---|---|
| TLS | Self-signed CA | Let's Encrypt / Vault PKI |
| Secrets | `kubectl create secret` | ESO + AWS SSM / HashiCorp Vault |
| Backstage image | Official `backstage/backstage` | Custom image with compiled plugins |
| Auth | GitHub OAuth | OIDC (Okta / Azure AD) |
| Git source | GitHub.com | Self-hosted Gitea / GitLab |
