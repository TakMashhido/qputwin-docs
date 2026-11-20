---
title: Quick start
description: Submit ideal, noisy, routed and asynchronous OpenQASM jobs to QPUTwin's quantum digital twins with curl and Python.
---

# Quick start

All examples assume the stack is running (`docker compose up -d --wait`) and listening on `localhost:8000`.

## 1. Run a Bell pair (ideal)

=== "curl"

    ```bash
    curl -s localhost:8000/v1/jobs -H 'content-type: application/json' -d '{
      "backend": "ionq_forte",
      "qasm": "OPENQASM 2.0;\ninclude \"qelib1.inc\";\nqreg q[2];\ncreg c[2];\nh q[0];\ncx q[0],q[1];\nmeasure q -> c;",
      "shots": 1000,
      "seed": 7
    }' | python3 -m json.tool
    ```

=== "Python (httpx)"

    ```python
    import httpx

    BELL = """OPENQASM 2.0;
    include "qelib1.inc";
    qreg q[2]; creg c[2];
    h q[0]; cx q[0], q[1];
    measure q -> c;
    """
    r = httpx.post("http://localhost:8000/v1/jobs",
                   json={"backend": "ionq_forte", "qasm": BELL, "shots": 1000, "seed": 7})
    body = r.json()
    print(body["result"]["counts"])                      # {'00': 5.., '11': 4..}
    print(body["compilation"]["native_gate_histogram"])  # {'gpi2': .., 'ms': 1, ...}
    print(body["telemetry"]["total_roundtrip_ms"])
    ```

The response has four parts:

| Part | Contents |
|---|---|
| `result` | `counts`, the echoed `seed`, `cregs`, and optionally `memory`, `statevector` and `density_matrix` |
| `compilation` | native gate histogram, depth, two-qubit depth, simulated width, `max_synthesis_error`, `hardware_time_estimate` |
| `telemetry` | additive latency decomposition (see [Telemetry](../guide/telemetry.md)) |
| headers | `Server-Timing`, `X-QPUTwin-Job-Id` |

## 2. Add the device's calibrated noise

```json
{
  "backend": "iqm_emerald",
  "qasm": "...Bell...",
  "shots": 10000,
  "seed": 7,
  "options": { "noise": { "model": "device" } }
}
```

You now see `01` and `10` outcomes, and `result.noise` explains the run: which engine was used, the per-gate depolarising strength, the estimated success probability, the purity, and **which calibration values were published and which were assumed**. See [Noise models](../guide/noise.md).

## 3. Get rejected by topology, then route around it

A star-shaped GHZ state (one hub entangled with every other qubit) cannot run on a square lattice:

```bash
python3 - <<'EOF'
import httpx
star = 'OPENQASM 2.0;\ninclude "qelib1.inc";\nqreg q[6];\ncreg c[6];\nh q[0];\n' + \
       "".join(f"cx q[0],q[{k}];\n" for k in range(1, 6)) + "measure q -> c;\n"
job = {"backend": "rigetti_ankaa_3", "qasm": star, "shots": 1000}
r = httpx.post("http://localhost:8000/v1/jobs", json=job)
print(r.status_code, r.json()["error"]["type"], r.json()["error"]["message"])
r = httpx.post("http://localhost:8000/v1/jobs", json={**job, "options": {"routing": {"method": "sabre"}}})
print(r.status_code, r.json()["compilation"]["routing"]["swaps_inserted"], "SWAPs")
EOF
```

The first call returns **422 `TopologicalCompilationError`**, quoting the offending statement, its line and the uncoupled qubit pair. The second succeeds and reports the SWAP overhead. See [Qubit routing](../guide/routing.md).

## 4. Inspect the exact statevector

```json
{ "backend": "quantinuum_helios", "qasm": "...", "shots": 1,
  "options": { "return_statevector": true } }
```

`result.statevector` is a list of `[re, im]` pairs over `result.statevector_qubits`. The compiler is phase-exact, so all twins return the *same* statevector, global phase included, for a circuit they can all run.

## 5. Submit asynchronously and poll

```bash
JOB=$(curl -s 'localhost:8000/v1/jobs?mode=async' -H 'content-type: application/json' \
      -d '{"backend":"ionq_forte","qasm":"OPENQASM 2.0;\nqreg q[1];\ncreg c[1];\nmeasure q -> c;"}' \
      | python3 -c 'import json,sys; print(json.load(sys.stdin)["job_id"])')
curl -s "localhost:8000/v1/jobs/$JOB?wait=10" | python3 -m json.tool   # long-poll up to 10 s
```

The submission returns **202** with a `Location` header. Once the job finishes, polling returns exactly the body a synchronous call would have returned. You can also cancel a queued job with `DELETE /v1/jobs/{id}`. See [Asynchronous jobs](../guide/async-jobs.md).

## 6. Explore the devices

```bash
curl -s localhost:8000/v1/backends | python3 -m json.tool              # status, replicas, natives
curl -s localhost:8000/v1/backends/iqm_emerald | python3 -m json.tool  # full spec + calibration
```

Next: the [tutorial](tutorial.md) runs one circuit on four of the twins and interprets the differences.
