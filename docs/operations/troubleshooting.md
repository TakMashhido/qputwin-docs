---
title: Troubleshooting
description: Fix common QPUTwin problems — topology rejections, memory limits, slow noisy jobs, unreachable backends, queue errors and provenance failures.
---

# Troubleshooting

??? question "`TopologicalCompilationError` on Rigetti / IQM"
    The circuit uses a qubit pair that is not coupled on the lattice. `details.qubits` and `details.neighbors` show which pair. Choose one of these fixes:

    * re-index the circuit along coupled qubits (rows of the grid are nearest-neighbour chains);
    * add `"options": {"routing": {"method": "sabre"}}`;
    * run it on a trapped-ion backend.

??? question "`ResourceLimitError: ... at most N simulated qubits fit`"
    The dense state would not fit in the worker's free memory. In order of preference:

    * touch fewer qubits (only touched qubits are simulated);
    * use `"precision": "complex64"`;
    * raise `QPUTWIN_WORKER_MEM`;
    * for noisy runs, let `engine: "auto"` choose trajectories instead of the density matrix.

    `details.max_feasible_qubits` reflects the memory free on the host *right now*.

??? question "A noisy job is slow"
    Check `result.noise.engine` and `telemetry.workload.simulated_qubits`:

    * The density matrix costs $4^n$, so prefer `engine: "trajectory"` beyond about 10 qubits (this is automatic under `auto`).
    * Trajectory cost is proportional to the trajectory count; lower `noise.trajectories` and accept `exact_sampling: false`.

??? question "`BackendUnavailableError` (503)"
    No worker replica accepted the connection. Check `docker compose ps` and `curl localhost:8000/v1/backends` (replica status). Workers may still be starting: `/v1/backends` fetches the device lazily once they come up.

??? question "`QueueFullError` (503)"
    More than `QPUTWIN_ASYNC_MAX_QUEUED` jobs are waiting. Back off, add worker replicas, or raise the limit.

??? question "`JobNotFoundError` when polling"
    The job expired (`QPUTWIN_ASYNC_TTL_S`, default 1 h), or, with API keys, it belongs to another client.

??? question "`CalibrationProvenanceError` (422)"
    The worker runs in strict provenance mode, and the job would use values tagged `assumed`. Load a published snapshot ([Calibration data](../guide/calibration.md)), override the listed parameters in `options.noise`, or disable the term that uses them.

??? question "Latency numbers vary between runs"
    Benchmarks are sensitive to host load. Check `uptime` and close heavy processes, keep `OMP_NUM_THREADS=1`, keep access logging off, warm up first, and report medians and percentiles, not single calls.

??? question "Statevectors differ between devices by a phase (QASM 2 `rzz`)"
    See [OpenQASM support](../guide/openqasm.md#libraries-and-vendor-natives). Write phase-sensitive comparisons in OpenQASM 3 (`stdgates.inc`).

??? question "The orchestrator cannot write `./data`"
    Set `QPUTWIN_UID`/`QPUTWIN_GID` in `.env` to the owner of the directory.
