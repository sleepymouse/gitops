# motd Service Setup

## Files created

```
gitops-repo/
├── applications/dev/motd.yaml          ← ArgoCD Application manifest
├── charts/motd/
│   ├── Chart.yaml
│   ├── values.yaml                     ← base defaults (port 8080, GHCR image)
│   └── templates/
│       ├── deployment.yaml             ← Spring Boot deployment with liveness/readiness probes
│       └── service.yaml                ← ClusterIP on port 8080
└── environments/dev/motd-values.yaml   ← dev overrides
```

The Deployment references Spring Boot Actuator health endpoints (`/actuator/health/liveness` and `/actuator/health/readiness`) — ensure your service has `spring-boot-starter-actuator` on the classpath.

## Registering the repo and bootstrapping ArgoCD

Run these steps in order. Requires a GitHub PAT with `repo` scope set as `$GITHUB_PAT`.

```bash
# 1. Register the repo as a secret in the argocd namespace
kubectl create secret generic gitops-repo \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/sleepymouse/gitops.git \
  --from-literal=username=sleepymouse \
  --from-literal=password=$GITHUB_PAT

# Label it so ArgoCD recognises it as a repo credential
kubectl label secret gitops-repo \
  --namespace argocd \
  argocd.argoproj.io/secret-type=repository

# 2. Apply the AppProject
kubectl apply -f gitops-repo/bootstrap/projects.yaml

# 3. Apply the root app
kubectl apply -f gitops-repo/bootstrap/root-app.yaml
```

Watch ArgoCD pick it up:

```bash
kubectl get applications -n argocd
```

## One-time prerequisite: create the imagePullSecret

The `ghcr-credentials` secret must exist in the namespace before the pod can pull the image. Run this once after ArgoCD creates the namespace (the `CreateNamespace=true` syncOption handles that):

```bash
kubectl create secret docker-registry ghcr-credentials \
  --docker-server=ghcr.io \
  --docker-username=sleepymouse \
  --docker-password=$CR_PAT \
  --namespace=app-motd-dev
```

## Note on valueFiles path

The `../../environments/dev/motd-values.yaml` path (relative to `charts/motd`) requires ArgoCD's repo-server to allow out-of-bounds value files. If sync fails with a path error, add `--enable-helm-value-files-out-of-bounds` to your ArgoCD repo-server args, or set `server.helm.valueFileSchemes` in the ArgoCD config.
