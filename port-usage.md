# Port Usage Reference

## Infrastructure

| Port | Protocol | Component | Purpose |
|------|----------|-----------|---------|
| 80   | HTTP     | ingress-nginx | HTTP ingress |
| 443  | HTTPS    | ingress-nginx | HTTPS ingress |

## Observability

| Port | Protocol | Component | Purpose |
|------|----------|-----------|---------|
| 9000 | HTTP     | MinIO | S3 API — used by Mimir, Loki and Tempo for object storage |
| 9001 | HTTP     | MinIO | Web console UI |
| 4317 | gRPC     | Alloy | OTLP gRPC receiver — accepts telemetry from Spring Boot apps |
| 4318 | HTTP     | Alloy | OTLP HTTP receiver — accepts telemetry from Spring Boot apps |
| 4317 | gRPC     | Tempo distributor | OTLP gRPC ingestion from Alloy |
| 3200 | HTTP     | Tempo query-frontend | Trace query API — used by Grafana datasource |
| 8080 | HTTP     | kube-state-metrics | Prometheus metrics scrape endpoint |
| 9100 | HTTP     | prometheus-node-exporter | Prometheus metrics scrape endpoint |

## Application Services

Allocated block: **8000–8099**

| Port      | Protocol | Component | Purpose                      |
|-----------|----------|-----------|------------------------------|
| 8000      | HTTP     | motd      | Spring Boot service          |
| 8001      | HTTP     | api       | Spring Boot service          |
| 8002      | HTTP     | auth      | Spring Boot service          |
| 8003      | HTTP     | frontend  | Spring Boot service          |
| 8004      | HTTP     | scheduler | Spring Boot service          |
| 8005      | HTTP     | worker    | Spring Boot service          |
| 8006-8099 | -        | reserved  | Future application services  |

## Notes

- Mimir and Loki are accessed internally via their nginx gateway on port 80 — no fixed external port required.
- Spring Boot services must not use port 9000 (MinIO S3 API) or 9001 (MinIO console).
- OTLP ports 4317 and 4318 are the OpenTelemetry standard and should be used consistently across all Spring Boot services.
