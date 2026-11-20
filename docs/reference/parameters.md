---
title: Request parameters
description: Every QPUTwin request parameter with type, default and allowed range — JobRequest, options, options.routing and options.noise.
---

# Request parameters

Unknown fields are rejected everywhere (`400 InvalidRequestError`).

## `JobRequest`

| Field | Type | Default | Range | Notes |
|---|---|---|---|---|
| `backend` | string | — (required) | registered ID | `quantinuum_helios` · `ionq_forte` · `rigetti_ankaa_3` · `iqm_emerald` |
| `qasm` | string | — (required) | 1 B … `QPUTWIN_MAX_QASM_BYTES` (1 MiB) | OpenQASM 2.0 / 3.x |
| `shots` | int | 1024 | 1 … 1 000 000 (`QPUTWIN_MAX_SHOTS`) | |
| `seed` | int \| null | null | 0 … 2⁶³−1 | null → server draws and echoes a seed |
| `priority` | int | 5 | 0 … 9 | async only; capped by the API key's max priority |
| `options` | object | `{}` | | below |

## `options`

| Field | Type | Default | Range / values | Constraints |
|---|---|---|---|---|
| `compilation` | string | `native` | `native`, `strict` | `strict` cannot be combined with routing |
| `return_statevector` | bool | false | | ideal engine; terminal-measurement circuits; ≤ `QPUTWIN_MAX_STATEVECTOR_QUBITS` (16) |
| `memory` | bool | false | | ≤ `QPUTWIN_MAX_MEMORY_SHOTS` (100 000) shots |
| `precision` | string \| null | worker default | `complex128`, `complex64` | |
| `routing` | object \| null | null | see below | |
| `noise` | object \| null | null | see below | |
| `engine` | string | `auto` | `auto`, `statevector`, `density_matrix`, `trajectory` | `trajectory` requires noise; noise forbids `statevector` |
| `return_density_matrix` | bool | false | | density-matrix engine; terminal circuits; ≤ `QPUTWIN_MAX_DENSITY_MATRIX_QUBITS` (8) |

## `options.routing`

| Field | Type | Default | Range |
|---|---|---|---|
| `method` | string | `sabre` | `sabre`, `none` |
| `initial_layout` | string \| int[] | `sabre` | `sabre`, `trivial`, or distinct non-negative ints |
| `exact_layout` | bool | true | |
| `exact_layout_budget` | int | 200000 | 0 … 10000000 |
| `trials` | int | 0 (automatic) | 0 … 64 |
| `layout_iterations` | int | 2 | 0 … 16 |
| `seed` | int | 0 | 0 … 2⁶³−1 |
| `lookahead` | int | 20 | 0 … 1000 |
| `lookahead_weight` | float | 0.5 | 0 … 10 |
| `decay` | float | 0.001 | 0 … 1 |
| `decay_reset` | int | 5 | 1 … 1000 |
| `bridge` | bool | false | |
| `optimize` | bool | true | |
| `noise_aware` | bool | true | |
| `noise_aware_budget` | int | 50000 | 0 … 10000000 |

## `options.noise`

| Field | Type | Default | Range |
|---|---|---|---|
| `model` | string | `device` | `device`, `ideal` |
| `scale` | float | 1.0 | 0 … 100 |
| `depolarizing`, `relaxation`, `idle`, `readout`, `coherent`, `crosstalk`, `reset_errors`, `correlated_readout`, `leakage`, `quasi_static` | bool | true | |
| `one_qubit_fidelity`, `two_qubit_fidelity`, `readout_fidelity`, `reset_fidelity` | float \| null | null (calibration) | 0.5 … 1 |
| `t1_us`, `t2_us` | float \| null | null | > 0, T2 ≤ 2·T1 |
| `overrotation` | float \| null | null | −0.5 … 0.5 |
| `coherent_z_rad`, `crosstalk_zz_rad` | float \| null | null | −π … π |
| `leakage_rate` | float \| null | null | 0 … 0.5 |
| `seepage_rate` | float \| null | null | 0 … 1 |
| `quasi_static_khz` | float \| null | null | 0 … 10⁶ |
| `trajectories` | int \| null | min(shots, `QPUTWIN_MAX_TRAJECTORIES`) | 1 … 10⁶, ≤ `QPUTWIN_MAX_TRAJECTORIES` |

## Combination rules

| Combination | Result |
|---|---|
| `noise` + `engine: statevector` | 400 |
| `engine: trajectory` without `noise` | 400 |
| `return_statevector` with `noise` or `engine: density_matrix` | 400 (use `return_density_matrix`) |
| `return_density_matrix` + `engine: statevector`/`trajectory` | 400 |
| `leakage` or `quasi_static` active + `engine: density_matrix` | 400 (they are time-correlated) |
| `routing` + `compilation: strict` | 400 |
| `return_statevector` / `return_density_matrix` with mid-circuit measurement or feed-forward | 400 `UnsupportedFeatureError` |
