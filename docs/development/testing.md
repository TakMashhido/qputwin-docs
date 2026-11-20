---
title: Testing
description: Run and extend QPUTwin's test suites — unit and in-process integration tests, live-stack integration and benchmark tests, and reports.
---

# Testing

## Unit and in-process integration (616 tests)

```bash
uv run pytest                    # ~1 min, no Docker needed
uv run pytest tests/test_noise_extended.py -k ramsey -v
uv run pytest --junitxml=benchmarks/results/unit_tests.junit.xml
```

| File | Covers |
|---|---|
| `test_gates.py`, `test_qasm_parser.py`, `test_qasm_library.py` | gate closed forms, the parser, libraries, modifiers |
| `test_synthesis.py` | native compilation exactness on every vendor |
| `test_engine.py` | statevector kernels, sampling, branching |
| `test_noise.py`, `test_noise_extended.py` | density matrix, every noise term, trajectories versus exact |
| `test_routing.py` | SABRE equivalence, bridges, cancellation, defects |
| `test_calibration.py`, `test_braket_import.py` | calibration, scheduling, overrides, importers |
| `test_simulators.py` | topology, limits, memory guard, the single-buffer executor under a tight budget |
| `test_fused.py` | fused statevector and density matrix against per-gate kernels, two-buffer against single-buffer executor, peak memory |
| `test_trajectory_batched.py` | batched trajectories against the exact density matrix and the sequential walk, thread independence, the cost-model engine choice |
| `test_calibrated_layout.py` | calibration-aware re-layout: never more expensive, same SWAPs and ideal output, avoids Heron's poor qubit, respects pinned layouts |
| `test_orchestrator.py`, `test_async_jobs.py`, `test_job_store.py` | HTTP API, telemetry identities, async queue, durability, auth, metrics |

In-process integration wires the orchestrator to the workers through an `httpx` ASGI transport, with injectable latency and dead hosts, so queueing, failover and recovery are tested deterministically.

## Live stack (47 tests)

```bash
QPUTWIN_FORTE_REPLICAS=2 docker compose up -d --wait   # 2 replicas so the scaling test runs
uv run pytest test_benchmark.py -v -s
```

`QPUTWIN_BASE_URL` (default `http://localhost:8000`) selects the stack, and `QPUTWIN_BENCH_REPS` (default 20) the benchmark repetitions. If the stack is unreachable, the suite **fails** with a clear message rather than skipping silently.

## Writing tests

* For physics, compare against an analytic result or an independent reference (`tests/conftest.py` provides a full-unitary `tensordot` reference).
* For statistics, assert within 5σ of the binomial expectation, or use a χ² test with a fixed seed.
* For service behaviour, use the in-process `HostRouter`/`Router` transports from `tests/test_orchestrator.py` and `tests/test_async_jobs.py`.
