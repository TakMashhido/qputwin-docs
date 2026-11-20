---
title: Submitting jobs
description: The QPUTwin job request and response — backend, OpenQASM, shots, seed, priority and every option, with curl and Python examples.
---

# Submitting jobs

## The request

`POST /v1/jobs` with a JSON body:

```json
{
  "backend": "quantinuum_helios",
  "qasm": "OPENQASM 3.0; include \"stdgates.inc\"; qubit[2] q; bit[2] c; h q[0]; cx q[0], q[1]; c = measure q;",
  "shots": 1024,
  "seed": 42,
  "priority": 5,
  "options": {
    "compilation": "native",
    "return_statevector": false,
    "memory": false,
    "precision": null,
    "routing": null,
    "noise": null,
    "engine": "auto",
    "return_density_matrix": false
  }
}
```

| Field | Type | Default | Constraints | Meaning |
|---|---|---|---|---|
| `backend` | string | required | a registered ID | `quantinuum_helios`, `ionq_forte`, `rigetti_ankaa_3`, `iqm_emerald` |
| `qasm` | string | required | ≤ 1 MiB (`QPUTWIN_MAX_QASM_BYTES`) | OpenQASM 2.0 or 3.x source, see [OpenQASM support](openqasm.md) |
| `shots` | int | 1024 | 1 … 1 000 000 | number of measurement samples |
| `seed` | int or null | null | 0 … 2⁶³−1 | PCG64 seed. When null, the server draws one and **echoes it back**, so every run is reproducible |
| `priority` | int | 5 | 0 … 9 | asynchronous jobs only; 9 runs first, capped by your API key |
| `options` | object | defaults | — | see the table below and [Request parameters](../reference/parameters.md) |

### Options

| Option | Values | Default | Guide |
|---|---|---|---|
| `compilation` | `native` \| `strict` | `native` | [Native compilation](compilation.md) |
| `return_statevector` | bool | false | exact pre-measurement state (ideal, terminal-measurement circuits, ≤ 16 qubits) |
| `memory` | bool | false | per-shot outcomes (≤ 100 000 shots) |
| `precision` | `complex128` \| `complex64` \| null | worker default | [Simulation engines](engines.md#precision) |
| `routing` | null \| object | null | [Qubit routing](routing.md) |
| `noise` | null \| object | null | [Noise models](noise.md) |
| `engine` | `auto` \| `statevector` \| `density_matrix` \| `trajectory` | `auto` | [Simulation engines](engines.md) |
| `return_density_matrix` | bool | false | final density matrix (terminal circuits, ≤ 8 qubits) |

Unknown fields are rejected (`400 InvalidRequestError`), so typos never pass silently.

## Execution modes

| Mode | How | Response |
|---|---|---|
| synchronous (default) | `POST /v1/jobs` | `200` with the full result, or an error envelope |
| asynchronous | `POST /v1/jobs?mode=async` or header `Prefer: respond-async` | `202` + `Location`; poll `GET /v1/jobs/{id}` |

See [Asynchronous jobs](async-jobs.md).

## The response

```json
{
  "job_id": "6f1c…",
  "backend": "quantinuum_helios",
  "status": "COMPLETED",
  "result":      { "counts": {"00": 517, "11": 507}, "seed": 42, "shots": 1024, "num_clbits": 2,
                   "cregs": {"c": [0, 2]}, "memory": null, "statevector": null, "density_matrix": null,
                   "noise": null },
  "compilation": { "native_gate_histogram": {"u1q": 3, "zzmax": 1, "rz": 2}, "native_depth": 5,
                   "max_synthesis_error": 1.1e-16, "hardware_time_estimate": {...}, "routing": null, ... },
  "telemetry":   { "total_roundtrip_ms": 4.9, "orchestration_overhead_ms": 1.8, "qpu_compute_time_ms": 0.6,
                   "breakdown": {...}, "workload": {...}, "environment": {...} }
}
```

Every field is documented in [Response schema](../reference/responses.md).

### Errors

Errors use the same envelope with `status` set to `REJECTED` (4xx) or `FAILED` (5xx), plus `error` (`type`, `message` and `details`) and **always** a `telemetry` object, because rejection latency is itself a measurement.

```json
{
  "status": "REJECTED",
  "error": {
    "type": "TopologicalCompilationError",
    "message": "'cx q[0],q[2];' (line 7) requires a two-qubit interaction between physical qubits 0 and 2, which are not coupled on rigetti_ankaa_3 ...",
    "details": { "qubits": [0, 2], "line": 7, "statement": "cx q[0],q[2];", "adjacency_entry": 0,
                 "neighbors": {"0": [1, 12], "2": [1, 3, 14]} }
  },
  "telemetry": { "engine_invoked": false, "qpu_compute_time_ms": 0.0, ... }
}
```

See [Errors](../reference/errors.md) for every type and its `details`.

## Client examples

=== "Python (httpx)"

    ```python
    import httpx

    client = httpx.Client(base_url="http://localhost:8000", timeout=600)

    def run(backend: str, qasm: str, **kw) -> dict:
        r = client.post("/v1/jobs", json={"backend": backend, "qasm": qasm, **kw})
        body = r.json()
        if r.status_code != 200:
            raise RuntimeError(f"{body['error']['type']}: {body['error']['message']}")
        return body

    body = run("iqm_emerald", open("ghz.qasm").read(), shots=4096, seed=1,
               options={"noise": {"model": "device"}})
    ```

=== "curl"

    ```bash
    jq -n --arg qasm "$(cat ghz.qasm)" \
      '{backend:"iqm_emerald", qasm:$qasm, shots:4096, seed:1, options:{noise:{model:"device"}}}' |
    curl -s localhost:8000/v1/jobs -H 'content-type: application/json' -d @-
    ```

=== "With an API key"

    ```bash
    curl -s localhost:8000/v1/jobs -H "Authorization: Bearer $QPUTWIN_KEY" \
         -H 'content-type: application/json' -d @job.json
    ```

## Reproducibility

* The same `seed` produces the same counts, bit for bit.
* The routing RNG has its own seed (`options.routing.seed`, default 0), so routing decisions do not depend on the shot seed.
* Asynchronous retries after a crashed instance reuse the stored seed, so a re-executed job returns identical counts.
