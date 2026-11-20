---
title: Calibration file format
description: JSON format of QPUTwin calibration snapshots and coupling maps — gate fidelities and durations, coherence, readout, overrides, crosstalk, leakage, parallelism, QCCD timing and provenance.
---

# Calibration file format

## Calibration snapshot

All keys except `device` are optional; anything absent is simply not modelled.

```json
{
  "device": "iqm_emerald",
  "label": "IQM Emerald - snapshot 2026-10-20",
  "snapshot_date": "2026-10-20",
  "fidelity_convention": "average_gate_fidelity",
  "gates": {
    "prx": { "fidelity": 0.9993, "duration_ns": 40 },
    "rz":  { "fidelity": 1.0,    "duration_ns": 0 },
    "cz":  { "fidelity": 0.995,  "duration_ns": 60, "overrotation": 0.0, "coherent_z_rad": 0.0, "leakage": 0.0 }
  },
  "readout":   { "fidelity": 0.97, "duration_ns": 1500 },
  "reset":     { "duration_ns": 2000, "fidelity": 0.99 },
  "coherence": { "t1_us": 54.7, "t2_us": 16.6, "quasi_static_detuning_khz": 0 },
  "feedforward_latency_ns": null,
  "shot_overhead_us": null,
  "qubits": {
    "7": { "t1_us": 31.0, "t2_us": 12.0, "readout_fidelity": 0.93,
           "gates": { "prx": { "fidelity": 0.998 } } }
  },
  "couplers": { "6-7": { "cz": { "fidelity": 0.981, "duration_ns": 70 } } },
  "crosstalk": { "zz_rad_per_two_qubit_gate": 0.002, "pairs": { "3-4": 0.004 } },
  "leakage": { "seepage_per_gate": 0.01 },
  "readout_correlations": [
    { "qubits": [0, 1], "matrix": [[0.95,0.03,0.03,0.00],[0.02,0.95,0.00,0.03],[0.03,0.00,0.95,0.02],[0.00,0.02,0.02,0.95]] }
  ],
  "parallelism": { "max_parallel_one_qubit": null, "max_parallel_two_qubit": null },
  "qccd": { "transport_ns_per_two_qubit_gate": 0, "cooling_ns_per_two_qubit_gate": 0 },
  "provenance": {
    "gates.cz.fidelity": "published: median CZ 99.5% (IQM / Amazon Braket launch, 2025)",
    "gates.cz.duration_ns": "assumed: placeholder",
    "*": "fallback tag for any field not listed"
  },
  "sources": ["https://…"]
}
```

## Field reference

| Path | Type | Range | Used by |
|---|---|---|---|
| `fidelity_convention` | string | `average_gate_fidelity` (default) \| `process_fidelity` | conversion $F_\text{avg} = (dF_e+1)/(d+1)$ |
| `gates.<g>.fidelity` | float | (0, 1] | depolarising solve |
| `gates.<g>.duration_ns` | float | ≥ 0 | schedule, relaxation |
| `gates.<g>.overrotation` | float | −0.5 … 0.5 | coherent error $U^\varepsilon$ |
| `gates.<g>.coherent_z_rad` | float | −π … π | residual Z / ZZ |
| `gates.<g>.fsim_theta_rad`, `fsim_phi_rad` | float | −π … π | two-qubit gates: residual `PhasedFSim(θ, φ)` after the gate (swap angle, conditional phase), as Google publishes per coupler; a coherent part of the gate's total error |
| `gates.<g>.leakage` | float | 0 … 0.5 | leak probability per gate and qubit |
| `readout.fidelity` or `readout.p01`/`p10` | float | fidelity (0,1]; p ≤ 0.5 | readout confusion |
| `readout.duration_ns`, `reset.duration_ns` | float | ≥ 0 | schedule |
| `reset.fidelity` | float | (0, 1] | reset error |
| `coherence.t1_us`, `t2_us` | float | > 0, T2 ≤ 2·T1 | relaxation |
| `coherence.quasi_static_detuning_khz` | float | ≥ 0 | trajectory detuning |
| `qubits.<q>.{t1_us, t2_us, readout_fidelity, readout_p01, readout_p10, quasi_static_detuning_khz, gates}` | | as above | per-qubit overrides |
| `couplers.<a-b>.<g>.{fidelity, duration_ns, …}` | | as above | per-coupler overrides |
| `crosstalk.zz_rad_per_two_qubit_gate`, `crosstalk.pairs.<a-b>` | float | −π … π | spectator ZZ |
| `leakage.seepage_per_gate` | float | 0 … 1 | leakage return |
| `readout_correlations[]` | {qubits, matrix} | ≥ 2 distinct qubits; column-stochastic $2^k\times2^k$; disjoint groups | joint readout (bit k ↔ `qubits[k]`) |
| `parallelism.max_parallel_one_qubit`, `max_parallel_two_qubit` | int | ≥ 1 | schedule |
| `qccd.transport_ns_per_two_qubit_gate`, `cooling_ns_per_two_qubit_gate` | float | ≥ 0 | schedule (idle before 2Q gates) |
| `feedforward_latency_ns`, `shot_overhead_us` | float | ≥ 0 | schedule, job-time estimate |
| `provenance` | {path: tag} | tag starts with `published` or `assumed` | provenance reports; most specific prefix wins; `*` = whole file |

Invalid files fail fast at worker start-up with a precise `ValueError`.

## Coupling map

```json
{
  "device": "rigetti_ankaa_3",
  "kind": "graph",
  "n_qubits": 84,
  "layout": { "rows": 7, "cols": 12, "labels": { "0": "0", "1": "1" } },
  "provenance": "Rigetti Ankaa-3 calibration 2026-10-01",
  "disabled_qubits": [5, 30],
  "edges": [[0, 1], [0, 12], [1, 2]]
}
```

| Field | Meaning |
|---|---|
| `kind` | `all_to_all` (no `edges` needed), `grid`, or `graph` |
| `n_qubits` | device size; an override map fixes `n_physical` |
| `edges` | undirected couplers `[p, q]` |
| `disabled_qubits` | defective qubits: they lose their couplers and any operation on them is rejected |
| `layout` | free-form metadata (`rows`/`cols` for grids, vendor `labels`) |
| `provenance` | quoted in error messages |

A replacement map must keep the device's connectivity class: an all-to-all device stays all-to-all.
