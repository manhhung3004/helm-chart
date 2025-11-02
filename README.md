
 
# Helm Charts Repository

This repository contains multiple Helm charts used across the project (applications, infrastructure, and third-party charts). The goal is to collect internal and customized charts in a single place to make installation, testing and deployment to Kubernetes easier.

## Table of contents

- Introduction
- Main directories and available charts
- Requirements
- Quick install (install/upgrade/uninstall)
- Developing and testing charts
- Configuration (values)
- Contributing
- License

## Introduction

This repo holds the primary Helm charts for the system, including charts for frontend, backend, ingress, Argo CD, Kong, MongoDB, Rancher, and supporting charts.

Top-level directories (examples):

- `argo-cd/` - Chart and configuration for Argo CD and related components.
- `Backend-Gentsshop/` - Chart for the Gentsshop backend service.
- `Frontend-Gentsshop/` - Chart for the Gentsshop frontend service.
- `ingress-nginx/` - Chart for ingress-nginx.
- `kong/` - Chart for Kong API Gateway.
- `mongodb/` - Chart for MongoDB.
- `rancher/` - Charts related to Rancher.
- `bento-social/` - Kubernetes manifests and docs for the `bento-social` service.

Each chart typically contains:

- `Chart.yaml` — chart metadata
- `values.yaml` — default values
- `templates/` — Kubernetes manifest templates

## Requirements

- Helm 3.x
- kubectl (to inspect cluster / namespaces)
- A Kubernetes cluster (local: kind/minikube, or cloud)

## Quick install

Example: installing Argo CD from this repository (run from repo root):

1) Create namespace and install:

```powershell
kubectl create namespace argocd -o yaml --dry-run=client | kubectl apply -f -
helm install argocd ./argo-cd -n argocd -f ./argo-cd/values.yaml
```

2) Upgrade chart:

```powershell
helm upgrade argocd ./argo-cd -n argocd -f ./argo-cd/values.yaml
```

3) Uninstall:

```powershell
helm uninstall argocd -n argocd
kubectl delete namespace argocd
```

Replace `argocd` and `./argo-cd` with the release name and chart path you want to install (for example `Backend-Gentsshop`, `Frontend-Gentsshop`, `kong`, ...).

### Installing with custom values

Create a `my-values.yaml` file with your overrides, then:

```powershell
helm install my-release ./Backend-Gentsshop -n my-namespace -f my-values.yaml
```

## Developing and testing charts

- Render templates locally to inspect generated manifests:

```powershell
helm template my-release ./Frontend-Gentsshop -f ./Frontend-Gentsshop/values.yaml
```

- Lint a chart:

```powershell
helm lint ./Backend-Gentsshop
```

- Run unit/integration tests if the chart provides them (see `templates/tests/` inside a chart).

## Configuration (values)

- Each chart provides a `values.yaml` file with defaults. Override values by passing `-f my-values.yaml` at install/upgrade time.
- If a chart has subcharts, check the `charts/` folder inside the chart for dependencies and their values.

Common configuration keys you may encounter:

- `replicaCount` — number of replicas for Deployments
- `image.repository` / `image.tag` — container image and tag
- `service.type` / `service.port` — service type and ports
- `ingress.enabled` / `ingress.hosts` — ingress configuration

## Operational tips

- Always run `helm lint` before installing to catch template issues.
- Use the `helm diff` plugin to preview changes before upgrading production releases:

```powershell
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade my-release ./chart -f values.yaml
```

- For production deployments, keep separate values files per environment: `values-prod.yaml`, `values-staging.yaml`, etc.

## Contributing

1. Fork the repository.
2. Create a feature or bugfix branch.
3. Validate changes with `helm lint` and `helm template`.
4. Open a pull request describing the change, rationale, and test steps.

Try to keep changes backward-compatible. If a change alters behavior or API, document it clearly in the PR.

## License

Check for a `LICENSE` file at repository root. If none exists, contact the repository owner to confirm licensing.

---

If you want, I can also:

- Add per-chart installation examples (release name, namespace, and dependencies).
- Add CI / lint badges if you have a CI workflow in this repo.
- Add a Troubleshooting section with common debugging commands (`kubectl logs`, `kubectl describe`, `helm history`, `helm rollback`).

Tell me which of the above you'd like me to add next, or if you prefer a bilingual (EN/VI) README.
