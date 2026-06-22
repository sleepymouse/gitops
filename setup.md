# motd Service Setup

## Files created

```
gitops-repo/
├── applications/dev/motd.yaml          ← ArgoCD Application manifest
├── charts/motd/
│   ├── Chart.yaml
│   ├── values.yaml                     ← base defaults (port 8080 (Tomcat default), GHCR image)
│   └── templates/
│       ├── deployment.yaml             ← Spring Boot deployment with liveness/readiness probes
│       └── service.yaml                ← ClusterIP on port 8080 (Tomcat default)
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

## Automated image tag updates (CI → GitOps)

The motd CI workflow (`.github/workflows/build-publish.yml`) automatically updates `environments/dev/motd-values.yaml` with the new image tag after each successful build. This requires a `GITOPS_PAT` secret in the motd repo so the workflow can write back to the gitops repo.

### Creating the GITOPS_PAT

1. Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Create a token with `repo` scope
3. Copy the token value

### Adding the secret to the motd repo

1. Go to `https://github.com/sleepymouse/motd` → Settings → Secrets and variables → Actions
2. Click **New repository secret**
3. Name: `GITOPS_PAT`
4. Value: paste the token created above
5. Click **Add secret**

Once in place, every push to `main` in the motd repo will commit the new image tag to the gitops repo and ArgoCD will resync automatically.

## Useful commands

Force ArgoCD to sync immediately rather than waiting for the 3-minute poll interval:

```bash
kubectl annotate application motd-dev -n argocd argocd.argoproj.io/refresh=hard
```

Watch pod status:

```bash
kubectl get pods -n app-motd-dev -w
```

Check which image a pod is running:

```bash
kubectl get pod -n app-motd-dev -o jsonpath='{.items[*].spec.containers[*].image}'
```

Check logs:

```bash
kubectl logs -n app-motd-dev -l app=motd-dev --tail=50
```

## Note on valueFiles path

The `../../environments/dev/motd-values.yaml` path (relative to `charts/motd`) requires ArgoCD's repo-server to allow out-of-bounds value files. If sync fails with a path error, add `--enable-helm-value-files-out-of-bounds` to your ArgoCD repo-server args, or set `server.helm.valueFileSchemes` in the ArgoCD config.
