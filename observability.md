# Observability Platform Installation Guide

## Purpose

This guide covers only the components deployed via Argo CD.

## Components

- Grafana
- MinIO
- Mimir (distributed)
- Loki (distributed)
- Tempo (distributed)
- Alloy
- kube-state-metrics
- prometheus-node-exporter

## Repository Structure

```text
platform-infra/
├── argocd/
├── observability/
│   ├── grafana/
│   ├── minio/
│   ├── mimir/
│   ├── loki/
│   ├── tempo/
│   ├── alloy/
│   ├── kube-state-metrics/
│   └── node-exporter/
└── docs/
```

## Prerequisites

### Host inotify limits

kind nodes are containers sharing the host kernel's inotify accounting — it is not namespaced per-container. A full LGTM stack (Loki, Mimir, Tempo, Alloy, Grafana) runs enough pods with fsnotify-based config/cert watchers to exceed the Linux default `fs.inotify.max_user_instances` (128) quickly; on this cluster usage reached ~1823 instances.

Once exhausted, the failure is **not confined to observability pods**. `kube-proxy` on the affected node crashes with `too many open files`, which breaks Service/ClusterIP and DNS routing for every other pod on that node — so ArgoCD, ingress-nginx, MetalLB, and any observability component scheduled there will crash-loop too, each with a different-looking symptom (i/o timeouts to the apiserver, DNS resolution failures, memberlist gossip failures, etc.).

Raise the limits on the actual Docker/kind host before installing (must be run on the host, not via `docker exec` into a node — the setting is shared, but only persists if applied at the host level):

```bash
sudo tee /etc/sysctl.d/99-kind-inotify.conf <<'CONF'
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches = 1048576
CONF
sudo sysctl --system
```

If pods are already crash-looping with `too many open files`, check current usage against the limit:

```bash
sudo sh -c 'grep -c "^inotify" /proc/[0-9]*/fdinfo/* 2>/dev/null | awk -F: "{s+=\$2} END{print s}"'
```

If usage is close to or above `max_user_instances`, raise the limit further and re-apply.

## Installation Order

1. Namespace and AppProject
2. MinIO
3. Mimir
4. Loki
5. Tempo
6. Grafana
7. kube-state-metrics
8. node-exporter
9. Alloy

## Argo CD Project

Create an AppProject named 'observability' and target the 'observability' namespace.

## Components

### MinIO
Helm chart: minio/minio

Responsibilities:
- S3-compatible object storage backend for Mimir, Loki and Tempo
- Provides the same API as AWS S3 so backend config is identical between dev and prod

### Mimir
Helm chart: grafana/mimir-distributed

Responsibilities:
- Metric storage
- Long-term retention
- PromQL query support

### Loki
Helm chart: grafana/loki (distributed mode)

Responsibilities:
- Log storage
- LogQL queries
- Kubernetes log aggregation

#### Retention

Configured in `gitops-repo/observability/loki/values/dev.yaml` under the `loki:` key:

```yaml
loki:
  limits_config:
    retention_period: 336h   # 14 days

  compactor:
    retention_enabled: true
    delete_request_store: s3
```

- `limits_config.retention_period` sets the global retention window (logs older than this are eligible for deletion).
- `compactor.retention_enabled` must be `true` for the compactor to actually enforce retention — otherwise `retention_period` is recorded but never acted on.
- `compactor.delete_request_store` must match `storage.type`/`object_store` (here `s3`, backed by MinIO) — the compactor needs somewhere to persist delete request markers.
- The compactor workload itself (top-level `compactor.replicas` in the same values file) must be `>= 1` — retention deletion is performed by the compactor's regular compaction cycle, no separate job is needed.
- Verify a values change renders correctly before syncing: `helm template loki grafana/loki --version <chart-version> -f gitops-repo/observability/loki/values/dev.yaml | grep -A3 retention_period`

#### Zone Awareness

The `grafana/loki` chart defaults to `ingester.zoneAwareReplication.enabled: true`, which enforces hard pod anti-affinity requiring each zone's ingester (`zone-a`, `zone-b`, `zone-c`) to run on a distinct node. On a small dev cluster (e.g. kind with one control-plane node and one worker node), only one zone can ever be scheduled — the rest sit `Pending` with `didn't match pod anti-affinity rules` / `untolerated taint`.

Since this dev config already runs with `commonConfig.replication_factor: 1` (no real redundancy expected), zone-awareness provides no benefit here and is disabled in `gitops-repo/observability/loki/values/dev.yaml`:

```yaml
ingester:
  replicas: 1
  zoneAwareReplication:
    enabled: false
```

Note `ingester:` is a top-level values key in this chart (alongside `distributor:`, `querier:`, etc.), not nested under `loki:`.

With this disabled, the chart renders a single `loki-ingester` StatefulSet instead of `loki-ingester-zone-a/b/c`. Confirm with:

```bash
helm template loki grafana/loki --version <chart-version> -f gitops-repo/observability/loki/values/dev.yaml | grep -E "^kind: StatefulSet$|name: loki-ingester"
```

### Tempo
Helm chart: grafana/tempo-distributed

Responsibilities:
- Distributed trace storage
- Trace search
- Service dependency analysis

### Grafana
Helm chart: grafana/grafana

Responsibilities:
- Dashboards
- Alerting
- Metrics, logs and traces exploration

Configure data sources:
- Mimir
- Loki
- Tempo

### kube-state-metrics
Helm chart: prometheus-community/kube-state-metrics

Provides:
- Deployment status
- Replica counts
- Pod state
- Job state
- HPA metrics

### prometheus-node-exporter
Helm chart: prometheus-node-exporter

Provides:
- CPU metrics
- Memory metrics
- Disk metrics
- Network metrics

### Alloy
Helm chart: grafana/alloy

Responsibilities:
- OTLP ingestion
- Log collection
- Metric scraping
- Forwarding to Mimir, Loki and Tempo

## Validation

Verify:

- All Argo CD applications are Healthy and Synced
- Grafana data sources connect successfully
- Metrics appear in Mimir
- Logs appear in Loki
- Traces appear in Tempo

## Initial Alerting

Create alerts for:

- Service unavailable
- High error rate
- High latency
- Pod restart storm
- Node memory pressure
- Node disk pressure

## Helm Chart Version Management

Before installing or upgrading, verify the pinned chart versions in `gitops-repo/observability/apps/` are valid.

### Add Helm repos

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add minio https://charts.min.io/
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Check latest available versions

```bash
helm search repo grafana/mimir-distributed
helm search repo grafana/loki
helm search repo grafana/tempo-distributed
helm search repo grafana/grafana
helm search repo grafana/alloy
helm search repo minio/minio
helm search repo prometheus-community/kube-state-metrics
helm search repo prometheus-community/prometheus-node-exporter
```

### Validate values against a chart version

Before syncing, dry-run a template render to catch any values key mismatches:

```bash
helm template <release-name> <repo>/<chart> \
  --version <chart-version> \
  -f gitops-repo/observability/<component>/values/dev.yaml
```

Major version bumps often include breaking changes to values key names. Check the chart changelog with:

```bash
helm show values <repo>/<chart> --version <chart-version>
```

## Production Tasks

- Configure SSO for Grafana
- Replace MinIO with real S3 object storage (update endpoint and credentials in Mimir, Loki and Tempo config)
- Configure retention policies
- Configure backups
- Store secrets outside Git
- Pin Helm chart versions