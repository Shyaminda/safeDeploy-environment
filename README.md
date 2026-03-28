# safeDeploy-environment

GitOps environment repository for deploying `demo-app` to Kubernetes with **Argo CD** and **Argo Rollouts**.

This repo defines:
- An Argo CD `Application` that points to this repository path.
- A canary rollout strategy for `demo-app`.
- Stable and canary Kubernetes Services used by Argo Rollouts.
- A Kustomize entrypoint for applying app manifests.

## 📁 Repository structure

```text
.
├── apps
│   ├── argocd
│   │   └── demo-app.yml          # Argo CD Application
│   └── demo-app
│       ├── kustomization.yaml    # Kustomize manifest list
│       ├── rollout.yaml          # Argo Rollouts canary strategy
│       ├── service-stable.yaml   # Stable service
│       └── service-canary.yaml   # Canary service
└── README.md
```

## 🚀 What is deployed

### 1) Argo CD Application (`apps/argocd/demo-app.yml`)
Creates an Argo CD `Application` named `demo-app` in namespace `argocd`.

Key behavior:
- Pulls manifests from this GitHub repo.
- Uses `apps/demo-app` as the source path.
- Deploys to the `default` namespace in the in-cluster Kubernetes API server.
- Auto-sync is enabled with `prune` and `selfHeal`.

### 2) Rollout (`apps/demo-app/rollout.yaml`)
Defines an Argo Rollout for `demo-app` with:
- `replicas: 2`
- Container image from GHCR (`ghcr.io/shyaminda/demo-app:55fccf8`)
- `ghcr-pull` image pull secret
- Canary strategy:
  - Shift 10% traffic to canary
  - Pause for manual/automated verification

### 3) Services (`apps/demo-app/service-stable.yaml`, `apps/demo-app/service-canary.yaml`)
Two services are used by Argo Rollouts:
- `demo-app-stable`
- `demo-app-canary`

Both expose port `3000` and select pods labeled `app: demo-app`.

### 4) Kustomize (`apps/demo-app/kustomization.yaml`)
Kustomize resource list includes:
- `rollout.yaml`
- `service-stable.yaml`
- `service-canary.yaml`

## ✅ Prerequisites

- A Kubernetes cluster
- Argo CD installed
- Argo Rollouts controller installed
- `kubectl` configured to access the cluster
- (Optional) Argo CD CLI and kubectl plugin for Argo Rollouts

## ⚙️ How to deploy

### Option A: Deploy through Argo CD (recommended) 🧭

1. Apply the Argo CD application manifest:
   ```bash
   kubectl apply -f apps/argocd/demo-app.yml
   ```
2. Argo CD will sync `apps/demo-app` automatically.

### Option B: Apply app manifests directly with kubectl 🛠️

```bash
kubectl apply -k apps/demo-app
```

> Note: Applying directly bypasses Argo CD as the source of truth. In GitOps workflows, prefer Option A.

## 📈 Rollout operations

### Check rollout status

```bash
kubectl argo rollouts get rollout demo-app -n default
```

### Promote rollout after pause

```bash
kubectl argo rollouts promote demo-app -n default
```

### Abort rollout

```bash
kubectl argo rollouts abort demo-app -n default
```

## 🔄 Updating the deployed version

To release a new app version:

1. Update the image tag in `apps/demo-app/rollout.yaml`.
2. Commit and push changes to this repository.
3. Let Argo CD sync (or manually sync via Argo CD UI/CLI).
4. Observe canary progression and pause step.
5. Promote when validation is complete.

## 🛡️ Rollback workflow (branch-based)

When a deployment issue is found, rollback should target the **functionally validated stable commit/tag** (the version that was verified in runtime), then be delivered through a **new rollback branch** for traceability.

1. Identify the stable functional version first (from release notes, incident timeline, Argo Rollouts/Argo CD history, or the currently stable image tag in cluster), then inspect commits:
   ```bash
   git log --oneline --decorate -n 20
   ```
2. Create a rollback branch from the current branch (example naming):
   ```bash
   git checkout -b rollback-incident-<timestamp>
   ```
3. Update `apps/demo-app/rollout.yaml` to the stable functional image tag (for example, `55fccf8`).
4. Commit the rollback change:
   ```bash
   git add apps/demo-app/rollout.yaml
   git commit -m "Rollback to image <known-good-tag>"
   ```
5. Open a PR and merge so Argo CD syncs the rollback through GitOps.

This repository history already includes multiple rollback branches and rollback commits, which can be used as references for naming and process.

## 📝 Notes

- Current Argo CD source branch is `main`.
- Current target namespace for workloads is `default`.
- `ghcr-pull` must exist in the deployment namespace for private GHCR image pulls.

## 🧪 Troubleshooting quick checks

```bash
kubectl get applications -n argocd
kubectl get rollout demo-app -n default
kubectl get svc demo-app-stable demo-app-canary -n default
kubectl describe rollout demo-app -n default
```

## 📄 License

No license file is currently defined in this repository.
