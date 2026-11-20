---
title: Validation
description: How QPUTwin's physics and system behaviour are validated — 616 unit and in-process tests and 47 live-stack tests, with the exact properties each proves.
---

# Validation

| Suite | Command | Tests | Result (2026-11-10) |
|---|---|---|---|
| Unit + in-process integration | `uv run pytest` | 616 | all pass |
| Live stack | `uv run pytest test_benchmark.py -v -s` | 47 | all pass (with 2 Forte replicas) |

JUnit reports are saved in `benchmarks/results/*.junit.xml`.

## Physics

| Property | How it is proved |
|---|---|
| Gate library | every closed form equals `scipy.linalg.expm` of its generator; unitarity to 1e-12 |
| Libraries | `stdgates.inc` and the vendor-native libraries are phase-exact; `qelib1.inc` is exact up to global phase |
| Modifiers | `inv`, `pow` and `ctrl` are exact (multi-control via Barenco Lemma 7.9) |
| Native compilation | random circuits on every vendor: native unitary = source unitary (global phase included); synthesis error < 1e-12 |
| Statevector kernels | agree with an independent `tensordot` reference (complex128 1e-12, complex64 2e-5) |
| Fused execution | fused statevector and fused superoperator density matrix equal the per-gate reference kernels to < 1e-12 on random circuits for all seven devices, block widths 1–5 and 1 or 3 threads, including every large-state branch (chunked gathers, cached and per-row phase tables) forced on small states; lazy activation keeps untouched qubits out of the state; the marginal read from the permuted state equals the reference; the two-level sampler draws the same inverse-CDF outcomes and never an empty bin; dropping trailing diagonals leaves every probability unchanged |
| Cross-simulator | ideal distributions of nine circuit families equal Qiskit Aer, Braket, Cirq and Qulacs to ≤ 2e-14 at 4–16 qubits ([comparison](simulator-comparison.md)) |
| Sampling | χ² goodness of fit; seed reproducibility; teleportation with feed-forward (4 branches) |
| Density-matrix kernels | unitary, Kraus, depolarising, reset and projection all equal an explicit dense reference |
| Error budget | the composite channel $\mathcal D_p\circ\mathcal R\circ\mathcal V$ hits the target average fidelity on a Choi state to 1e-10 |
| Noise terms | single-gate error = 1 − F; T1 decay during gates and idle windows; over-rotation sin²(πε/2); crosstalk sin²(θ/2); reset and correlated readout; leakage statistics; Gaussian Ramsey; fixed-timeline idle slots; all against analytic formulas within 5σ |
| Trajectories | agree with the exact density matrix (TV < 0.03) with every Markovian term on, including mid-circuit feed-forward and reset |
| Routing | routed state = source state relabelled by the final layout, to 1e-9 including phase, on random circuits, with and without bridges; embeddable circuits get an exact SWAP-free layout; the kept SABRE trial has the fewest emitted two-qubit gates |
| Scheduling | parallelism, barriers, feed-forward, aligned read-out, QCCD transport, missing durations, and that the O(N) interval pruning reproduces the exhaustive schedule exactly |

## System

| Property | How it is proved |
|---|---|
| Topology enforcement | adjacency-matrix properties; star-GHZ rejected with the offending pair recomputed independently; chain accepted |
| Resource model | caps, memory guard (`max_feasible_qubits` actually fits), engine fallbacks |
| Calibration | packaged files cover every native gate; provenance tags on every value; overrides, importer, Braket import |
| Telemetry | the additive identities on every response, including errors and asynchronous jobs |
| Async queue | queueing, cancel, long-poll, retention, priority, fair share (including round-robin between clients on a single dispatch slot), global capacity across two instances, restart survival, lease recovery and abandonment |
| Replicas | least-in-flight balancing, connect-failover, DNS discovery |
| Security & ops | API-key isolation and priority caps, `/metrics`, `/ready` |

## Manual drills

The HA path was exercised by hand against the live `ha` profile:

* a graceful stop of one of two orchestrators lost no jobs;
* a `SIGKILL` mid-job was recovered by lease expiry, with the re-execution returning identical counts.
