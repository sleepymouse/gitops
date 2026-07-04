# Developer Guide

## Purpose

This guide describes the changes required in Spring Boot 4 services to integrate with the observability platform.

## Dependencies

Gradle:

```gradle
dependencies {
    implementation "org.springframework.boot:spring-boot-starter-actuator"
    implementation "org.springframework.boot:spring-boot-starter-opentelemetry"
}
```

## application.properties

```properties
spring.application.name=customer-service

management.endpoint.health.probes.enabled=true
management.endpoints.web.exposure.include=health,info,metrics

management.opentelemetry.resource-attributes.service.name=${spring.application.name}
management.opentelemetry.resource-attributes.service.namespace=ecrebo
management.opentelemetry.resource-attributes.deployment.environment=${ENVIRONMENT:dev}

management.otlp.metrics.export.enabled=true
management.otlp.metrics.export.url=http://alloy.observability.svc.cluster.local:4318/v1/metrics

management.otlp.tracing.endpoint=http://alloy.observability.svc.cluster.local:4318/v1/traces
```

## Kubernetes Configuration

Add:

```yaml
env:
  - name: ENVIRONMENT
    value: production
```

## Health Probes

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
```

## Structured Logging

Log JSON to stdout.

Recommended fields:

- timestamp
- level
- service
- environment
- trace_id
- span_id
- logger
- thread
- message

Recommended dependency:

```gradle
implementation "net.logstash.logback:logstash-logback-encoder"
```

## Business Metrics

Use Micrometer.

Example metrics:

- receipts.processed
- receipts.failed
- offers.generated
- offers.failed
- till.requests
- till.failures

Good metric tags:

- service
- environment
- channel
- region
- result

Avoid:

- user_id
- request_id
- receipt_id
- basket_id
- email

## Tracing

Ensure trace_id and span_id are present in logs.

This enables:

```text
Metrics -> Traces -> Logs
```

## Developer Checklist

- Actuator enabled
- OpenTelemetry enabled
- Service name configured
- Health probes configured
- JSON logging configured
- Trace IDs present in logs
- Business metrics implemented where appropriate
