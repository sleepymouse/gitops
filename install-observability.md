# Installing the Observability Stack

This guide covers standing up the full LGTM observability stack (MinIO, Mimir, Loki, Tempo, Grafana, Alloy) on a dev cluster and validating it end-to-end.

## Prerequisites

- kind cluster running
- ArgoCD installed in the `argocd` namespace
- This repo cloned and pushed to GitHub (ArgoCD pulls from `https://github.com/sleepymouse/gitops.git`)

## 1. Bootstrap ArgoCD

Apply the root app and AppProject. These are not managed by ArgoCD itself — they must be applied manually.

```bash
kubectl apply -f gitops-repo/bootstrap/observability-project.yaml
kubectl apply -f gitops-repo/bootstrap/root-app.yaml
```

The root app watches `gitops-repo/applications/dev` and will auto-create the `observability` app-of-apps, which in turn deploys all stack components.

## 2. Wait for components to deploy

The stack deploys in sync waves to ensure dependencies are ready before dependents start:

| Wave | Components |
|------|------------|
| -3   | MinIO |
| -2   | Mimir, Loki, Tempo |
| -1   | Grafana |
|  0   | kube-state-metrics, node-exporter |
|  1   | Alloy |

Monitor progress:

```bash
kubectl get pods -n observability -w
```

Full readiness takes 3–5 minutes. All pods should reach `1/1 Running` or `2/2 Running`. Expected pod count is approximately 30.

> **Note:** On a single-node kind cluster, `loki-ingester-zone-b-0` and `loki-ingester-zone-c-0` will remain `Pending` — this is expected. Loki runs with zone-aware replication but only zone-a can schedule on a single node. Loki still functions correctly with zone-a alone.

If any pods are stuck, check ArgoCD sync status — see the [Debugging](#debugging) section at the end of this guide.

## 3. Access Grafana

Port-forward the Grafana service:

```bash
kubectl port-forward -n observability svc/grafana 3000:80
```

Open `http://localhost:3000` and log in with `admin` / `admin`.

## 4. Validate datasources

Datasources are provisioned via Helm values and are **read-only in the Grafana UI**. If a test fails and you need to change an endpoint, edit `gitops-repo/observability/grafana/values/dev.yaml` and commit — do not try to edit them in the browser.

Go to **Connections → Data Sources** and click **Test** on each datasource:

- **Mimir** — should return green
- **Loki** — should return green
- **Tempo** — should return green

## 5. Validate telemetry

**Metrics (Mimir)**

Go to **Explore**, select the **Mimir** datasource, switch to Code mode, and run:

```
up
```

You should see time series from `kube-state-metrics` and `node-exporter`.

**Logs (Loki)**

Go to **Explore**, select **Loki**, and run:

```
{namespace="observability"}
```

You should see log streams from the observability pods.

**Traces (Tempo)**

Go to **Explore**, select **Tempo**, and use the **Search** tab to look for recent traces. Traces will only appear once a Spring Boot service instrumented with OpenTelemetry is sending data through Alloy.

## Troubleshooting

**Mimir pods crashing with "bucket does not exist"**

The Mimir Helm chart includes a bundled MinIO subchart (`mimir-minio`) which is enabled by default. When enabled, it overrides the storage configuration and causes Mimir to connect to `mimir-minio` instead of the standalone MinIO — which doesn't have the right buckets. The values already have `minio.enabled: false` to prevent this, but if you see this error verify that setting is in place in `gitops-repo/observability/mimir/values/dev.yaml`.

If the setting is correct but the error persists, the standalone MinIO's bucket creation job may not have run. Exec into the MinIO pod to create the buckets manually:

```bash
MINIO_POD=$(kubectl get pod -n observability -l app=minio -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n observability $MINIO_POD -- /bin/bash -c \
  "mc alias set local http://localhost:9000 minioadmin minioadmin && \
   mc mb local/mimir-blocks && \
   mc mb local/mimir-ruler && \
   mc mb local/mimir-alertmanager"
```

Then delete the crashing pods to force an immediate restart:

```bash
kubectl delete pod -n observability mimir-ingester-0 mimir-compactor-0 \
  mimir-alertmanager-0 mimir-store-gateway-0 mimir-ruler-0
```

**Alloy sync error — CustomResourceDefinition not permitted**

If the `alloy` ArgoCD app fails with a CRD permission error, the AppProject change may not have been applied yet:

```bash
kubectl apply -f gitops-repo/bootstrap/observability-project.yaml
argocd app sync alloy
```

**ArgoCD sync stuck with "another operation is already in progress"**

```bash
argocd app terminate-op <app-name>
argocd app sync <app-name> --prune
```

---

## Debugging

### Install the ArgoCD CLI

```bash
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

### Access the ArgoCD UI

Port-forward the ArgoCD server:

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

Open `https://localhost:8080`. Username is `admin`, get the password with:

```bash
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Log in via CLI

```bash
argocd login localhost:8080 --insecure
```

### ArgoCD commands

```bash
# List all applications and their sync/health status
argocd app list

# Show detailed status and resource list for an app
argocd app get <app-name>

# Manually trigger a sync
argocd app sync <app-name>

# Sync and remove resources no longer in git
argocd app sync <app-name> --prune

# Kill a stuck sync operation
argocd app terminate-op <app-name>

# Stream live logs from an app's sync operation
argocd app logs <app-name>
```

> **Note:** The AppProject (`observability-project.yaml`) and root app (`root-app.yaml`) are not managed by ArgoCD — changes to these must be applied with `kubectl apply`, not `argocd app sync`.

### Kubernetes commands

**Pod status**

```bash
# Watch all observability pods
kubectl get pods -n observability -w

# Filter to a specific component
kubectl get pods -n observability | grep mimir
kubectl get pods -n observability | grep loki
kubectl get pods -n observability | grep tempo
kubectl get pods -n observability | grep alloy
```

**Logs**

```bash
# Logs from a running pod
kubectl logs -n observability <pod-name>

# Logs from the previous (crashed) container instance
kubectl logs -n observability <pod-name> --previous

# Tail logs from a deployment
kubectl logs -n observability deployment/<name> --tail=50

# Stream logs from all pods with a label
kubectl logs -n observability -l app.kubernetes.io/name=alloy -f
```

**Describe a pod** (shows events, resource limits, restart reasons)

```bash
kubectl describe pod -n observability <pod-name>
```

**Services**

```bash
# List all services in the observability namespace
kubectl get svc -n observability

# Check a specific service's ports
kubectl get svc -n observability <service-name>
```

**Jobs** (useful for checking MinIO bucket creation)

```bash
kubectl get jobs -n observability
kubectl logs -n observability job/<job-name>
```

**Port-forwarding to internal services**

```bash
# Grafana
kubectl port-forward -n observability svc/grafana 3000:80

# MinIO S3 API
kubectl port-forward -n observability svc/minio 9000:9000

# ArgoCD
kubectl port-forward -n argocd svc/argocd-server 8080:443
```
