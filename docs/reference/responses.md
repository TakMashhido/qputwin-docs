---
title: Response schema
description: Field-by-field reference of QPUTwin job responses — result, compilation, routing, hardware time estimate, noise report, workload and telemetry.
---

# Response schema

## Envelope

| Field | Present | Meaning |
|---|---|---|
| `job_id` | always | 32-hex identifier (also in `X-QPUTwin-Job-Id`) |
| `backend` | always | backend ID |
| `status` | always | `COMPLETED`, `REJECTED`, `FAILED` (sync); plus `QUEUED`, `RUNNING`, `CANCELLED` (async) |
| `result` | success | below |
| `compilation` | success | below |
| `error` | failure | `{type, message, details}`, see [Errors](errors.md) |
| `telemetry` | always | below |
| `execution_mode`, `client`, `priority`, `submitted_at`, `started_at`, `finished_at`, `http_status`, `attempts`, `links` | async | job metadata |

## `result`

| Field | Type | Meaning |
|---|---|---|
| `counts` | {bitstring: int} | histogram; bit strings print as `c[n-1]…c[0]` |
| `shots`, `seed` | int | the seed actually used (echoed even when drawn) |
| `num_clbits` | int | classical bits |
| `cregs` | {name: [offset, size]} | classical register layout |
| `memory` | [str] \| null | per-shot outcomes (`options.memory`) |
| `statevector` | [[re, im]] \| null | ideal pre-measurement state (`return_statevector`) |
| `statevector_qubits` | [int] \| null | the physical qubits the state covers (bit j ↔ qubit[j]) |
| `density_matrix` | [[[re, im]]] \| null | final ρ (`return_density_matrix`) |
| `noise` | object \| null | noise report ([Noise models](../guide/noise.md#the-noise-report-resultnoise)) |

## `compilation`

| Field | Meaning |
|---|---|
| `mode` | `native` or `strict` |
| `qasm_version` | `2.0`, `3.0`, … |
| `declared_qubits`, `physical_qubits_used`, `simulated_qubits`, `idle_measured_qubits` | qubit accounting |
| `input_gate_count`, `input_multi_qubit_gate_count` | source circuit after library expansion |
| `native_op_count`, `native_physical_op_count`, `native_virtual_op_count`, `native_two_qubit_count` | native gate counts |
| `measurement_count`, `reset_count` | |
| `native_depth`, `two_qubit_depth` | depth (virtual gates and barriers ignored) |
| `native_gate_histogram` | {gate: count} |
| `global_phase`, `max_synthesis_error` | phase-exactness bookkeeping and certificate |
| `routing` | routing report or null ([Qubit routing](../guide/routing.md#the-routing-report-compilationrouting)) |
| `hardware_time_estimate` | [gate-duration model](../guide/timing.md#the-report) |

## `telemetry`

| Field | Meaning |
|---|---|
| `schema_version` | `1.2` |
| `total_roundtrip_ms`, `orchestration_overhead_ms`, `qpu_compute_time_ms` | headline metrics |
| `engine_invoked` | false when the job was rejected before the engine ran |
| `execution_mode`, `queue_clock`, `attempt` | sync/async; clock used for queue time; execution attempt |
| `breakdown` | 16 additive spans ([Telemetry](../guide/telemetry.md)) |
| `workload` | `simulated_qubits`, `engine`, `precision`, `statevector_bytes`, `state_bytes`, `estimated_peak_bytes`, `memory_budget_bytes`, `max_sim_qubits`, `shots`, `execution_path`, `branches`, `native_op_count`, `native_two_qubit_count`, `noisy`, `routed` |
| `environment` | orchestrator version, Python, serving `replica`, and the worker environment (id, vendor, numpy, BLAS threads, CPU affinity, cgroup CPU limit, precision, cap, concurrency, coupling map, calibration) |
| `clock` | `perf_counter_ns` |
| `timestamp_utc` | wall-clock label only (never used for durations) |

`workload.execution_path` takes one of these values:

* `terminal_sampling`: ideal, single simulation;
* `shot_branching`: ideal, mid-circuit measurements;
* `density_matrix` / `density_matrix_branching`: noisy, exact;
* `trajectories`: noisy, sampled.
