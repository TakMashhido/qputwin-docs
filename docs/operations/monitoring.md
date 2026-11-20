---
title: Monitoring
description: Monitor QPUTwin with readiness probes and Prometheus metrics — job counters, latency histograms, queue depth, replica load and example alerts.
---

# Monitoring

!!! tip "Ready-made monitoring stack"
    The `monitoring/` folder runs Prometheus, Grafana, cAdvisor, Blackbox Exporter, Alertmanager and Portainer CE in Docker, with Node Exporter on the host. It adds per-job metrics from the telemetry dataset and six dashboards. See `monitoring/README.md`.

## Probes

| Endpoint | Use for | Healthy |
|---|---|---|
| `GET /health` | liveness | always 200 while the process runs |
| `GET /ready` | readiness / load-balancer checks | 200 when the job store answers and ≥ 1 backend has been reached; otherwise 503 |

## Prometheus metrics (`GET /metrics`)

| Metric | Type | Labels | Meaning |
|---|---|---|---|
| `qputwin_jobs_total` | counter | `backend`, `status`, `mode` | finished jobs (COMPLETED / REJECTED / FAILED / CANCELLED; sync / async) |
| `qputwin_job_roundtrip_ms` | histogram | `backend` | server round trip of completed jobs |
| `qputwin_job_orchestration_overhead_ms` | histogram | `backend` | orchestration overhead |
| `qputwin_job_qpu_compute_ms` | histogram | `backend` | engine time |
| `qputwin_async_jobs` | gauge | `backend`, `state` | queued / running async jobs (all instances) |
| `qputwin_async_capacity` | gauge | `backend` | dispatch slots |
| `qputwin_replica_in_flight` | gauge | `backend`, `replica` | in-flight requests (this instance) |
| `qputwin_replica_dispatched_total` | counter | `backend`, `replica` | requests sent |
| `qputwin_replica_failures_total` | counter | `backend`, `replica` | connection failures |
| `qputwin_build_info` | gauge | `version`, `instance` | build and instance identity |

The metrics are fed from the same telemetry stream as the research dataset, so the two can never disagree.

### Scrape configuration

```yaml
scrape_configs:
  - job_name: qputwin
    metrics_path: /metrics
    static_configs:
      - targets: ["qputwin-host:8000"]
```

### Example alerts

```yaml
groups:
  - name: qputwin
    rules:
      - alert: QPUTwinBackendFailing
        expr: rate(qputwin_jobs_total{status="FAILED"}[5m]) > 0
        for: 5m
      - alert: QPUTwinQueueBacklog
        expr: qputwin_async_jobs{state="queued"} > 10 * qputwin_async_capacity
        for: 10m
      - alert: QPUTwinSlowJobs
        expr: histogram_quantile(0.95, sum by (le, backend) (rate(qputwin_job_roundtrip_ms_bucket[5m]))) > 1000
        for: 10m
      - alert: QPUTwinReplicaDown
        expr: increase(qputwin_replica_failures_total[5m]) > 0
```

## Research-grade telemetry

For latency studies, use the JSONL dataset (`./data/telemetry.jsonl`) or `GET /v1/telemetry/summary` rather than Prometheus histograms. They carry the full additive span decomposition of every job; see [Telemetry](../guide/telemetry.md).
