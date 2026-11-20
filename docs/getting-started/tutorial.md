---
title: "Tutorial: four QPUs, one circuit"
description: Step-by-step tutorial comparing a GHZ circuit on trapped-ion and superconducting quantum digital twins with QPUTwin, covering topology, noise, timing and latency.
---

# Tutorial: four QPUs, one circuit

This tutorial runs a 6-qubit GHZ state on four of the seven digital twins (Helios, Forte, Ankaa-3 and Emerald) and reads the four reports QPUTwin returns: compilation, noise, timing and telemetry. It takes about 20 minutes.

## The circuit

A GHZ state $\frac{1}{\sqrt 2}(|0\dots0\rangle + |1\dots1\rangle)$ built along a **nearest-neighbour path**. On the trapped-ion twins (all-to-all) the path is simply `q[0]…q[5]`. On a square lattice it must follow coupled qubits; along the first row of the grid it is `q[0]…q[5]` too, so one QASM program fits every device:

```python
GHZ6 = """OPENQASM 2.0;
include "qelib1.inc";
qreg q[6]; creg c[6];
h q[0];
cx q[0],q[1]; cx q[1],q[2]; cx q[2],q[3]; cx q[3],q[4]; cx q[4],q[5];
measure q -> c;
"""
```

## Step 1: run it ideal and noisy everywhere

```python
import httpx

API = "http://localhost:8000"
BACKENDS = ["quantinuum_helios", "ionq_forte", "rigetti_ankaa_3", "iqm_emerald"]

for b in BACKENDS:
    ideal = httpx.post(f"{API}/v1/jobs", json={"backend": b, "qasm": GHZ6, "shots": 4000, "seed": 1}).json()
    noisy = httpx.post(f"{API}/v1/jobs", json={"backend": b, "qasm": GHZ6, "shots": 20000, "seed": 1,
                                               "options": {"noise": {"model": "device"}}}).json()
    c = noisy["result"]["counts"]
    success = (c.get("000000", 0) + c.get("111111", 0)) / 20000
    comp = ideal["compilation"]
    print(f"{b:18s} natives={comp['native_gate_histogram']} "
          f"2Q-depth={comp['two_qubit_depth']} "
          f"t_shot={comp['hardware_time_estimate']['circuit_duration_us']:.2f} us "
          f"GHZ success={success:.3f} "
          f"ESP={noisy['result']['noise']['estimated_success_probability']:.3f}")
```

## Step 2: read the compilation report

* **Native gates differ per vendor.** Helios uses `u1q` and `zzmax`, Forte `gpi2`, `gpi` and `ms`, Ankaa-3 `rx` and `cz`, and Emerald `prx` and `cz`. Each CX becomes **exactly one** native entangler on every device.
* `max_synthesis_error` is ~1e-15: the native circuit reproduces the source unitary to machine precision, global phase included.
* `two_qubit_depth` is 5 everywhere, because the GHZ chain is inherently sequential.

## Step 3: read the noise report

`result.noise` contains:

* `engine`: `density_matrix` for 6 qubits (exact). Wider circuits switch to `trajectory` automatically.
* `per_gate`: the published fidelity of each gate type and the depolarising strength `p` solved from it.
* `estimated_success_probability`: the product of all gate and readout fidelities, a quick predictor.
* `parameters_used.published` and `assumed_parameters`: on the superconducting twins, gate durations and readout fidelity are typically *assumed*, while T1/T2 and 2Q fidelities are *published*.

!!! example "What you should observe"
    Helios keeps GHZ success near 99% and Forte near 95% (its SPAM error of 0.5% per qubit adds up). The superconducting twins land near 75–80%, and the [error budget](../research/benchmarks.md#error-budget) shows why: readout (≈3% per qubit, assumed) dominates, followed by T1 decay of the $|1\dots1\rangle$ branch and gate depolarisation.

## Step 4: read the timing report

`hardware_time_estimate.circuit_duration_us` is about **2 µs** per shot on the superconducting twins and **0.7–4 ms** on the trapped-ion twins, a gap of three orders of magnitude. IonQ Forte serialises entangling gates on its shared motional bus (`parallelism.max_parallel_two_qubit = 1`). `estimated_job_duration_s` multiplies by the shot count and flags that per-shot overhead is excluded, because no vendor publishes it.

## Step 5: read the telemetry

```python
t = noisy["telemetry"]
print(t["total_roundtrip_ms"], t["orchestration_overhead_ms"], t["qpu_compute_time_ms"])
print(t["breakdown"])
```

The identity `total_roundtrip_ms = orchestration_overhead_ms + breakdown.worker_service_ms` holds exactly, and so does `qpu_compute_time_ms = simulation_ms + sampling_ms`. The same numbers are appended to `./data/telemetry.jsonl` for later analysis.

## Step 6: break the topology on purpose

Replace the chain with a star (`cx q[0],q[k]` for every k): the trapped-ion twins accept it, the superconducting twins reject it with `TopologicalCompilationError`, and with `"routing": {"method": "sabre"}` they accept it again at the cost of extra CZ gates. The [routing benchmark](../research/benchmarks.md#cost-of-routing) quantifies that cost as width grows.
