---
title: HTTP API
description: QPUTwin HTTP API reference — /v1/jobs, /v1/backends, /v1/telemetry, /health, /ready and /metrics on the orchestrator, plus the internal worker API.
---

# HTTP API

Base URL: `http://<host>:8000` (or the HA gateway on `:8080`). Request and response bodies are JSON. When API keys are configured, `/v1/*` endpoints require `Authorization: Bearer <key>` or `X-API-Key: <key>` ([Security](../operations/security.md)).

## Orchestrator endpoints

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/health` | no | liveness |
| `GET` | `/ready` | no | readiness: job store answers and ≥ 1 backend reached (`503` otherwise) |
| `GET` | `/metrics` | no | Prometheus metrics (`text/plain; version=0.0.4`) |
| `GET` | `/` | no | service info and contact |
| `GET` | `/v1/backends` | yes | every backend's status and summary |
| `GET` | `/v1/backends/{id}` | yes | full device specification (`?refresh=true` re-fetches) |
| `POST` | `/v1/jobs` | yes | submit a job (sync by default; `?mode=async` → `202`) |
| `GET` | `/v1/jobs` | yes | list asynchronous jobs and queue statistics |
| `GET` | `/v1/jobs/{id}` | yes | job status or result (`?wait=s` long-poll) |
| `DELETE` | `/v1/jobs/{id}` | yes | cancel a queued job |
| `GET` | `/v1/telemetry` | yes | recent telemetry records (`?limit=&backend=`) |
| `GET` | `/v1/telemetry/summary` | yes | per-backend descriptive statistics |

### `POST /v1/jobs`

| Query / header | Values | Effect |
|---|---|---|
| `mode` | `sync` (default) \| `async` | asynchronous returns `202` + `Location` |
| `Prefer: respond-async` | header | same as `mode=async` |
| `X-QPUTwin-Client` | string ≤ 64 chars | client name for fair share (ignored when API keys are on) |

Body: see [Request parameters](parameters.md). Responses:

* `200`: the result ([Response schema](responses.md));
* `202`: accepted for asynchronous execution;
* `4xx`/`5xx`: the error envelope ([Errors](errors.md)).

Response headers: `Server-Timing`, `X-QPUTwin-Job-Id`, plus `Location` and `Retry-After` on `202`.

### `GET /v1/jobs/{id}`

| Query | Default | Meaning |
|---|---|---|
| `wait` | 0 | seconds to wait for completion (capped by `QPUTWIN_MAX_POLL_WAIT_S`, default 30) |

Returns `200` with the summary (`QUEUED`/`RUNNING`, including `queue_position`) or the full result body plus timestamps (`COMPLETED`/`REJECTED`/`FAILED`/`CANCELLED`). Returns `404 JobNotFoundError` for an unknown, expired or (with authentication) foreign job.

### `DELETE /v1/jobs/{id}`

Cancels a `QUEUED` job and returns its body with `status: "CANCELLED"`. Any other state returns `409 JobStateError`.

### `GET /v1/jobs`

| Query | Meaning |
|---|---|
| `status` | `QUEUED`, `RUNNING`, `COMPLETED`, `REJECTED`, `FAILED`, `CANCELLED` |
| `backend` | backend ID |
| `limit` | 1–10 000 (default 100) |

Returns `{"jobs": [summary, …], "queues": {backend: {"queued", "running", "capacity"}}}`.

### `GET /v1/backends`

Returns `{"backends": [...]}`. Each item has `id`, `status` (`online`/`offline`/`http_<code>`), `vendor`, `name`, `technology`, `n_physical`, `connectivity`, `native_gates`, `max_sim_qubits`, `replicas` (url, status, in_flight, dispatched, failures), `disabled_qubits` and `calibration` (label).

### `GET /v1/backends/{id}`

The full `DeviceSpec` plus emulator limits:

* `coupling_map`: kind, n_edges, degree, edges, disabled_qubits, layout, provenance;
* `native_gates`, `entangler`, `synthesis_basis`, `param_constraints`;
* `calibration`, with `provenance`;
* `emulator`: max_sim_qubits, precision, memory_guard, available_memory_bytes, max_shots, max_statevector_qubits, max_density_matrix_qubits, engines, max_trajectories, density_auto_max_qubits, require_published_calibration, routing.

### `GET /v1/telemetry/summary`

```json
{"summary": {"iqm_emerald": {"jobs": 412, "completed": 401, "rejected": 11, "failed": 0, "cancelled": 0,
  "async": 24, "metrics": {"total_roundtrip_ms": {"n": 401, "mean": …, "stdev": …, "min": …,
  "median": …, "p95": …, "max": …}, "orchestration_overhead_ms": {…}, "qpu_compute_time_ms": {…}}}}}
```

## Worker endpoints (internal network only)

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/health` | liveness + vendor |
| `GET` | `/v1/device` | full device specification |
| `POST` | `/v1/execute` | run a job: `{qasm, shots, seed, options, job_id}`; returns `result`, `compilation`, `workload`, `timing` (durations), `worker` (environment) |

Workers are not reachable from the host in the Compose deployment. Every benchmarked path goes through the orchestrator.
