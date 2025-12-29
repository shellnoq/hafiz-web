---
title: Monitoring
description: Monitoring Hafiz with Prometheus and Grafana
---

# Monitoring

## Prometheus Metrics

Hafiz exposes metrics at `/metrics` on the main S3 port (9000 by default).

### Access Metrics

```bash
curl http://localhost:9000/metrics
```

### Key Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `hafiz_requests_total` | Counter | Total requests |
| `hafiz_request_duration_seconds` | Histogram | Request latency |
| `hafiz_objects_total` | Gauge | Object count |
| `hafiz_storage_bytes` | Gauge | Storage used |
| `hafiz_active_connections` | Gauge | Active connections |

### Prometheus Config

```yaml
scrape_configs:
  - job_name: 'hafiz'
    static_configs:
      - targets: ['hafiz:9000']
    metrics_path: '/metrics'
```

## Grafana Dashboard

Import the dashboard from:
```
deploy/grafana/dashboards/hafiz.json
```

## Alerts

Example PrometheusRule:

```yaml
groups:
  - name: hafiz
    rules:
      - alert: HafizHighErrorRate
        expr: rate(hafiz_requests_total{status="error"}[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
```

## Health Checks

```bash
# Health
curl http://localhost:9000/health

# Readiness
curl http://localhost:9000/ready

# Liveness
curl http://localhost:9000/live
```
