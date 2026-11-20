---
title: Project layout
description: QPUTwin repository structure — the qputwin_core physics package, orchestrator and worker services, scripts, configuration, benchmarks and documentation.
---

# Project layout

```text
pyproject.toml / uv.lock        uv-managed project (extras: engine, worker, orchestrator; groups: dev, bench)
LICENSE / NOTICE                Apache-2.0 license and copyright notice
config/backends.toml            orchestrator backend registry
config/devices/                 calibrated coupling maps / calibration snapshots (mounted into workers)
docker/{orchestrator,worker}.Dockerfile, docker/gateway/nginx.conf
docker-compose.yml / .env       stack definition and per-container settings
src/qputwin_core/               shared physics + language core
  qasm/                         lexer, parser, expression evaluator, embedded libraries, flattener (pure Python)
  ir.py                         Operation / Circuit IR
  gates.py                      closed-form unitaries
  synthesis.py                  phase-exact native compiler
  topology.py                   CouplingMap (adjacency, distances, components, disabled qubits)
  devices.py                    DeviceSpec + the seven device specs, runtime overrides
  calibration.py                calibration model + provenance (pure Python)
  routing.py                    SABRE router, bridges, commutation-aware cancellation
  scheduling.py                 ASAP gate-duration model
  noise.py                      calibration-driven noise model
  engine.py density.py trajectory.py   statevector, density-matrix, trajectory engines
  simulators.py                 QPUSimulator, TrappedIonSimulator, SuperconductingSimulator
  devices_data/                 coupling maps + calibration/<device>.json
src/qputwin_orchestrator/       app, middleware, registry, jobs (durable queue), telemetry, metrics, schemas
src/qputwin_worker/             worker app factory + vendors/{quantinuum_helios, ionq_forte, rigetti_ankaa_3, iqm_emerald, ibm_heron, ibm_nighthawk, google_willow}.py
scripts/                        generate_coupling_maps.py, import_coupling_map.py, import_braket_device.py
tests/                          616 unit + in-process integration tests
test_benchmark.py               47 live-stack integration + benchmark tests
benchmarks/                     run_benchmarks.py (21 experiments), circuits.py, plot_figures.py, results/ (JSON + JUnit), figures/ (PDF/SVG/PNG + CSV)
```

The source code is in the [Cloudslab/QPUTwin](https://github.com/Cloudslab/QPUTwin) repository. This documentation site has its own repository, [qputwin-docs](https://github.com/TakMashhido/qputwin-docs).

## Dependency boundaries

* `qputwin_core.qasm`, `ir` and `calibration` are pure Python, so the orchestrator image ships without numpy.
* Everything else in `qputwin_core` uses numpy (and scipy for `logm`/`expm` in the coherent-error model).
* The worker depends on `qputwin_core`; the orchestrator depends only on its pure-Python parts.
