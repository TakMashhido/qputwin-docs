---
title: Errors
description: Every QPUTwin error type with its HTTP status, when it is raised and the details it carries — from QASM syntax errors to topology violations and queue limits.
---

# Errors

All errors share one envelope, which **always** includes `telemetry`:

```json
{ "job_id": "…", "backend": "…", "status": "REJECTED",
  "error": { "type": "…", "message": "…", "details": { … } },
  "telemetry": { "engine_invoked": false, … } }
```

`status` is `REJECTED` for 4xx and `FAILED` for 5xx. Error types travel unchanged from worker to client.

| HTTP | Type | Raised by | When | Key `details` |
|---|---|---|---|---|
| 400 | `QASMSyntaxError` | orchestrator / worker | malformed source | `line`, `col` |
| 400 | `QASMSemanticError` | orchestrator / worker | undeclared names, bad arity, out-of-range index, unknown include | `line` |
| 400 | `UnsupportedFeatureError` | orchestrator / worker | outside the supported OpenQASM 3 subset; `return_statevector` with mid-circuit measurement | `line` |
| 400 | `InvalidRequestError` | both | bad option values or combinations, payload too large, schema violations | `errors`, limits |
| 401 | `AuthenticationError` | orchestrator | missing or invalid API key (only when keys are configured) | — |
| 404 | `UnknownBackendError` | orchestrator | backend not registered | `available` |
| 404 | `JobNotFoundError` | orchestrator | unknown, expired or foreign async job | `job_id` |
| 409 | `JobStateError` | orchestrator | cancelling a job that is not `QUEUED` | `status` |
| 422 | `TopologicalCompilationError` | worker | uncoupled qubit pair, or a disabled qubit | `qubits`, `line`, `statement`, `neighbors`, `adjacency_entry`, `reason` |
| 422 | `NativeGateError` | worker | `strict` mode: non-native gate or off-grid angle | `gate`, `primitive`, `native_gates`, `allowed` |
| 422 | `CompilationError` | worker | no native lowering exists | `gate` |
| 422 | `ResourceLimitError` | both | more qubits than the device; over the simulation cap; memory guard; no connected region for routing | `required_bytes`, `available_bytes`, `max_feasible_qubits`, `max_sim_qubits` |
| 422 | `CalibrationProvenanceError` | worker | strict provenance mode and unpublished values used | `unpublished_parameters` |
| 500 | `QPUTwinError` | both | internal error (should not happen; please report) | — |
| 502 | `BackendProtocolError` | orchestrator | the worker returned a non-QPUTwin payload | — |
| 503 | `BackendUnavailableError` | orchestrator | no reachable replica (connection refused or DNS) | `replicas` |
| 503 | `QueueFullError` | orchestrator | `QPUTWIN_ASYNC_MAX_QUEUED` reached | `queued`, `max_queued` |
| 504 | `BackendTimeoutError` | orchestrator | the worker did not answer within `QPUTWIN_REQUEST_TIMEOUT_S` | — |
| 507 | (`QPUTwinError`) | worker | `MemoryError` despite the guard (defensive backstop) | — |

## Examples

These are real responses from the running stack. The memory-guard example reports the memory that was actually free on the host at the time.

=== "Topology"

    ```json
    {"type": "TopologicalCompilationError",
     "message": "'cx q[0],q[2];' (line 7) requires a two-qubit interaction between physical qubits 0 and 2, which are not coupled on rigetti_ankaa_3 (grid 7x12, nearest-neighbour only; no routing)",
     "details": {"device": "rigetti_ankaa_3", "gate": "cx", "primitive": "CX", "qubits": [0, 2], "line": 7,
                 "statement": "cx q[0],q[2];", "neighbors": {"0": [1, 12], "2": [1, 3, 14]}, "adjacency_entry": 0}}
    ```

=== "Memory guard"

    ```json
    {"type": "ResourceLimitError",
     "message": "a dense 36-qubit complex128 statevector run needs ~3.0 TiB but only 5.2 GiB is available to this worker; at most 26 simulated qubits fit (use complex64 precision, raise the container memory limit, or touch fewer qubits)",
     "details": {"simulated_qubits": 36, "required_bytes": 3298534883328, "available_bytes": 5617725440,
                 "max_feasible_qubits": 26, "precision": "complex128", "device": "ionq_forte", "engine": "statevector"}}
    ```

=== "Strict provenance"

    ```json
    {"type": "CalibrationProvenanceError",
     "details": {"unpublished_parameters": ["gates.cz.duration_ns", "gates.prx.duration_ns", "readout.fidelity"],
                 "device": "iqm_emerald"}}
    ```

## Handling errors in clients

* **4xx:** fix the request; retrying will not help.
* **409, 404 on poll:** check the job state or id.
* **503 `QueueFullError`:** back off and retry later.
* **503 `BackendUnavailableError`, 504:** transient; retry with backoff. Asynchronous jobs are retried automatically by the queue after an instance crash, but never after a worker timeout, because the worker may still be executing the job.
