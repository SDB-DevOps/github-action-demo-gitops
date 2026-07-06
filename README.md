# github-action-demo — GitOps

Desired-state manifests for [`github-action-demo`](https://github.com/sdb-devops/github-action-demo),
reconciled into the cluster by **ArgoCD**. This repo is the single source of
truth for what runs in the cluster — nothing is deployed by `kubectl` anymore.

## Flow

```
push to app repo
      │
      ▼
CI builds & pushes  ghcr.io/…/github-action-demo:sha-<short>
      │
      ▼
Deploy workflow bumps `newTag` here → opens a PR
      │
      ▼  (a human reviews & merges)
merge to main
      │
      ▼
ArgoCD auto-syncs the new image into the cluster
```

## Layout

```
apps/github-action-demo/   # Kustomize base: deployment, service, image tag
argocd/application.yaml    # ArgoCD Application that watches apps/github-action-demo
```

`apps/github-action-demo/kustomization.yaml` holds the deployed image tag under
`images[].newTag`. That field is the only thing the deploy workflow changes.

## One-time cluster setup

1. Install ArgoCD (if not already):

   ```sh
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. Point this repo's URL correctly in `argocd/application.yaml` (`spec.source.repoURL`),
   then register the Application:

   ```sh
   kubectl apply -f argocd/application.yaml
   ```

   If this GitOps repo is **private**, first give ArgoCD read access:

   ```sh
   argocd repo add https://github.com/sdb-devops/github-action-demo-gitops.git \
     --username <user> --password <token>
   ```

That's it — from here, merging a PR into `main` is what ships.

## Manual change / rollback

Edit `images[].newTag` (or run `kustomize edit set image ...` in
`apps/github-action-demo/`), commit, and merge. To roll back, set the tag to a
previous `sha-<short>` value. ArgoCD converges the cluster to match.
