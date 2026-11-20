---
title: Configuration
description: Every QPUTwin environment variable — worker simulation limits, memory guard, noise engines, orchestrator timeouts, job store, API keys, discovery, Docker Compose scaling and backends.toml.
---

# Configuration

Configuration comes from environment variables (Compose reads `.env`) and, for the orchestrator, `config/backends.toml`. Environment variables override the TOML.

## Worker

| Variable | Default | Meaning |
|---|---|---|
| `QPUTWIN_VENDOR` | — (required) | `quantinuum_helios`, `ionq_forte`, `rigetti_ankaa_3`, `iqm_emerald` |
| `QPUTWIN_HOST` / `QPUTWIN_PORT` | `0.0.0.0` / `8001` | bind address |
| `QPUTWIN_MAX_SIM_QUBITS` | device size | simulation-width cap for this container |
| `QPUTWIN_PRECISION` | `complex128` | `complex128` or `complex64` |
| `QPUTWIN_MEMORY_GUARD` | `1` | enable the memory-feasibility guard |
| `QPUTWIN_MEMORY_BUDGET_BYTES` | auto | override the detected memory budget |
| `QPUTWIN_MEMORY_RESERVE_BYTES` | 268435456 (256 MiB) | headroom kept free |
| `QPUTWIN_MAX_SHOTS` | 1000000 | shots per job |
| `QPUTWIN_MAX_MEMORY_SHOTS` | 100000 | shots when `options.memory` is on |
| `QPUTWIN_MAX_STATEVECTOR_QUBITS` | 16 | `return_statevector` limit |
| `QPUTWIN_MAX_DENSITY_MATRIX_QUBITS` | 8 | `return_density_matrix` limit |
| `QPUTWIN_DENSITY_AUTO_MAX_QUBITS` | 12 | widest noisy circuit `engine=auto` may send to the density matrix (it also needs $2^n \le 4T$; see [engines](../guide/engines.md#automatic-selection-engine-auto)) |
| `QPUTWIN_MAX_TRAJECTORIES` | 2000 | trajectory cap (default count = min(shots, cap)) |
| `QPUTWIN_MAX_OPS` | 5000000 | flattened-operation limit (guards against include bombs) |
| `QPUTWIN_REQUIRE_PUBLISHED_CALIBRATION` | `0` | strict provenance for noisy runs |
| `QPUTWIN_COUPLING_MAP` | packaged | path to a coupling-map JSON (runtime override) |
| `QPUTWIN_CALIBRATION` | packaged | path to a calibration JSON (runtime override) |
| `QPUTWIN_WORKER_CONCURRENCY` | `1` | concurrent jobs per worker (1 models a QPU's serial queue) |
| `QPUTWIN_ENGINE_THREADS` | worker CPU quota (at most 64) | threads the fused engine uses per job (`1` = single-threaded); BLAS stays at `OMP_NUM_THREADS=1` |
| `QPUTWIN_ENGINE_IN_PLACE` | `auto` | fused executor buffers: `auto` = two buffers when they fit, else one (peak 1.5 × state); `always`; `never` |
| `QPUTWIN_ENGINE_GROW_THREADS` | 4 | threads used while the stored state grows (lazy activation); more contend on page faults |
| `QPUTWIN_TRAJECTORY_WORKERS` | 4 | trajectory sub-batches run concurrently (more was measured slower: interpreter lock) |
| `QPUTWIN_TRAJECTORY_BATCHED` | `1` | `0` forces the sequential trajectory walk (for comparison) |
| `QPUTWIN_WORKER_ID` | hostname | identifier reported in telemetry |
| `QPUTWIN_LOG_LEVEL` / `QPUTWIN_ACCESS_LOG` | `info` / `0` | logging (access logs perturb latency) |
| `OMP_NUM_THREADS` | `1` (image) | BLAS threads; pinned for reproducibility |

## Orchestrator

| Variable | TOML key | Default | Meaning |
|---|---|---|---|
| `QPUTWIN_BACKENDS_CONFIG` | — | `config/backends.toml` | registry file |
| `QPUTWIN_BACKEND_URL_<ID>` | `backends.<id>.url(s)` | from TOML | worker URL(s); comma-separate replicas |
| `QPUTWIN_HOST` / `QPUTWIN_PORT` | — | `0.0.0.0` / `8000` | bind address |
| `QPUTWIN_REQUEST_TIMEOUT_S` | `request_timeout_s` | 600 | worker request timeout |
| `QPUTWIN_CONNECT_TIMEOUT_S` | `connect_timeout_s` | 5 | connect timeout |
| `QPUTWIN_MAX_QASM_BYTES` | `max_qasm_bytes` | 1048576 | request size limit |
| `QPUTWIN_MAX_SHOTS` | `max_shots` | 1000000 | shots limit |
| `QPUTWIN_TELEMETRY_BUFFER` | `telemetry_buffer` | 10000 | `/v1/telemetry` ring buffer |
| `QPUTWIN_TELEMETRY_PATH` | `telemetry_path` | unset (`/app/data/telemetry.jsonl` in Compose) | JSONL research dataset |
| `QPUTWIN_DISCOVERY` | `discovery` | `static` (`dns` in Compose) | replica discovery |
| `QPUTWIN_DISCOVERY_INTERVAL_S` | `discovery_interval_s` | 10 | DNS re-resolution interval |
| `QPUTWIN_JOB_STORE` | `job_store` | `memory` (`sqlite` in Compose) | async job store |
| `QPUTWIN_JOB_STORE_PATH` | `job_store_path` | unset (`/app/data/jobs.sqlite3`) | SQLite file |
| `QPUTWIN_JOB_LEASE_S` | `job_lease_s` | 30 | running-job lease |
| `QPUTWIN_JOB_MAX_ATTEMPTS` | `job_max_attempts` | 3 | executions before an abandoned job fails |
| `QPUTWIN_JOB_POLL_INTERVAL_S` | `job_poll_interval_s` | 0.2 | queue polling period |
| `QPUTWIN_ASYNC_MAX_QUEUED` | `async_max_queued` | 10000 | queued jobs before `QueueFullError` |
| `QPUTWIN_ASYNC_RETENTION` | `async_retention` | 10000 | finished jobs kept |
| `QPUTWIN_ASYNC_TTL_S` | `async_ttl_s` | 3600 | finished-job expiry |
| `QPUTWIN_ASYNC_PER_REPLICA_CONCURRENCY` | `async_per_replica_concurrency` | 1 | dispatch slots per worker replica |
| `QPUTWIN_MAX_POLL_WAIT_S` | `max_poll_wait_s` | 30 | long-poll cap |
| `QPUTWIN_API_KEYS` | — | empty | `name:key[:max_priority]`, comma-separated; enables authentication |
| `QPUTWIN_API_KEYS_FILE` | — | unset | file with one key spec per line (Docker secrets) |

## Docker Compose (`.env`)

| Variable | Default | Meaning |
|---|---|---|
| `QPUTWIN_<DEVICE>_MAX_QUBITS` | 98 / 36 / 84 / 54 | per-container cap (`DEVICE` ∈ `HELIOS`, `FORTE`, `ANKAA3`, `EMERALD`) |
| `QPUTWIN_<DEVICE>_REPLICAS` | 1 | worker replicas per QPU |
| `QPUTWIN_<DEVICE>_COUPLING_MAP` / `_CALIBRATION` | empty | runtime data files (paths inside the container) |
| `QPUTWIN_WORKER_CPUS` / `QPUTWIN_WORKER_MEM` | `4` / `12g` | identical resource limits for every worker |
| `QPUTWIN_ORCHESTRATOR_REPLICAS` | 1 | orchestrator replicas (HA) |
| `QPUTWIN_ORCHESTRATOR_PORTS` | `8000` | host port, or a range such as `8000-8002` for replicas |
| `QPUTWIN_GATEWAY_PORT` | 8080 | `ha`-profile gateway port |
| `QPUTWIN_UID` / `QPUTWIN_GID` | 1000 | owner of `./data` files |

## `config/backends.toml`

```toml
[orchestrator]
request_timeout_s = 600.0
connect_timeout_s = 5.0
max_qasm_bytes = 1048576
max_shots = 1000000
telemetry_buffer = 10000
discovery = "static"
discovery_interval_s = 10.0
async_max_queued = 10000
async_retention = 10000
async_ttl_s = 3600.0
async_per_replica_concurrency = 1
max_poll_wait_s = 30.0
job_store = "memory"
job_lease_s = 30.0
job_max_attempts = 3
job_poll_interval_s = 0.2

[backends.ionq_forte]
url = "http://worker-ionq-forte:8001"          # or: urls = ["http://a:8001", "http://b:8001"]
description = "IonQ Forte - trapped ion, 36 qubits, all-to-all"
```
