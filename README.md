# sigstore-gitops

GitOps repository for `sigstore-app`. Contains all Kubernetes manifests and ArgoCD application definitions. No application code lives here — only desired cluster state.

CI writes to this repo. ArgoCD reads from it. Nothing else touches the cluster directly.

---

## Repository Structure

```
sigstore-gitops/
├── apps/
│   ├── base/                        # env-agnostic base manifests
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── namespace.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/                     # auto-updated by CI on every main push
│       │   ├── kustomization.yaml   # image: main-<sha>
│       │   └── namespace.yaml
│       ├── staging/                 # auto-updated by CI on tag push
│       │   ├── kustomization.yaml   # image: v<semver>
│       │   ├── deployment-patch.yaml
│       │   └── namespace.yaml
│       └── prod/                    # updated via PR merge only
│           ├── kustomization.yaml   # image: v<semver>
│           ├── deployment-patch.yaml
│           └── namespace.yaml
├── argocd/
│   ├── apps-of-apps.yaml            # root ArgoCD app — bootstraps everything
│   ├── apps/
│   │   ├── dev-app.yaml             # auto-sync on
│   │   ├── staging-app.yaml         # auto-sync on
│   │   └── prod-app.yaml            # no auto-sync — human approves
│   └── infra/
│       └── sigstore-policy-controller.yaml
└── policies/
    └── sigstore-image-policy.yaml   # ClusterImagePolicy — rejects unsigned images
```

---

## How Promotion Works
### Dev: automatic on every main push

CI builds and signs the image, pushes it as `main-<sha>`, then commits the new image tag to `apps/overlays/dev/kustomization.yaml`. ArgoCD detects the change and syncs immediately.

```
git push origin main
  → CI builds + cosign signs → pushes main-<sha>
  → CI commits new tag to overlays/dev
  → ArgoCD auto-syncs dev
```

### Staging: automatic on tag push

Tagging a commit triggers CD. CI verifies the Cosign signature on `main-<sha>`, retags it as `v<semver>`, then commits the new tag to `apps/overlays/staging/kustomization.yaml`. ArgoCD auto-syncs staging.

```
git tag v1.0.0 && git push origin v1.0.0
  → CD verifies cosign signature
  → retags main-<sha> → v1.0.0 (no rebuild)
  → CI commits new tag to overlays/staging
  → ArgoCD auto-syncs staging
  → smoke tests run
  → CD opens PR to update overlays/prod
```

### Prod: manual PR approval

CD opens a pull request updating `apps/overlays/prod/kustomization.yaml`. A human reviews and merges. ArgoCD detects the diff and **waits for manual sync approval** in the ArgoCD UI — it does not auto-deploy to prod.

```
PR merged → overlays/prod updated
  → ArgoCD detects drift
  → human clicks Sync in ArgoCD UI
  → cluster admission webhook verifies cosign signature before pod starts
  → deployed
```

---

## Image Tagging Convention

| Environment | Tag format        | Example                          |
|-------------|-------------------|----------------------------------|
| dev         | `main-<sha>`      | `main-44c7492a98...`             |
| staging     | `v<semver>`       | `v1.0.0`                         |
| prod        | `v<semver>`       | `v1.0.0`                         |

The same image digest travels through all environments. **Images are never rebuilt for staging or prod**  the `main-<sha>` image built and signed by CI is retagged and promoted. The Cosign signature is on the digest, not the tag.

---

## ArgoCD Setup

### Bootstrap

``` bash
### Apply the Sigstore Policy Controller:

kubectl apply -f argocd/infra

### Apply the Sigstore policy for Sigstore App (my app)
kubectl apply -f policies/

### Apply the root app-of-apps to your cluster once:

kubectl apply -f argocd/apps-of-apps.yaml
```

ArgoCD will self-manage all child applications from that point forward.

### Sync Policy per Environment

| Environment | Auto-sync | Self-heal | Prune |
|-------------|-----------|-----------|-------|
| dev         | ✅        | ✅        | ✅    |
| staging     | ✅        | ✅        | ✅    |
| prod        | ❌        | ❌        | ✅    |

Prod requires a human to approve sync in the ArgoCD UI after the PR is merged.

---

## Signing Policy

All images from `docker.io/shilucloud/sigstore-app` must carry a valid Cosign signature issued by Fulcio, with the signing identity originating from this repository's GitHub Actions workflows.

The `ClusterImagePolicy` in `policies/sigstore-image-policy.yaml` enforces this at the cluster admission level  unsigned images are rejected before a pod can start, regardless of how they were deployed.


---

## Making Changes

**Never edit `apps/overlays/dev` manually**  CI owns this file and will overwrite it on the next main push.

**Never edit `apps/overlays/staging` manually**  CI owns this file and will overwrite it on the next tag push.

**`apps/overlays/prod` is only updated via PR**  direct commits to prod overlay are not permitted by convention. Always go through the PR process so there is a review record.

**Base manifests** (`apps/base/`) can be edited directly via PR. Changes propagate to all environments on next ArgoCD sync.

---

## Required Secrets (App Repo)

These secrets live in the **app repo** (`sigstore-app`), not here:

| Secret              | Purpose                                      |
|---------------------|----------------------------------------------|
| `DOCKERHUB_USERNAME` | Push images to Docker Hub                   |
| `DOCKERHUB_TOKEN`   | Docker Hub access token                      |
| `GITOPS_PAT`        | GitHub PAT with write access to this repo    |

---

## Related

- App repo: [github.com/shilucloud/sigstore-app](https://github.com/shilucloud/sigstore-app)
- Docker Hub: [hub.docker.com/r/shilucloud/sigstore-app](https://hub.docker.com/r/shilucloud/sigstore-app)