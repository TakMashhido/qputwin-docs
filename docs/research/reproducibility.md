---
title: Reproducibility
description: Reproducibility practices in QPUTwin — seeds, monotonic clocks, pinned builds, environment capture, calibration provenance and a checklist for reporting results.
---

# Reproducibility

| Practice | Implementation |
|---|---|
| **Seeds** | every run is seeded (PCG64) and the seed is always echoed back; routing has its own seed; async retries reuse the stored seed |
| **Clocks** | durations come from `perf_counter_ns` inside a single process; workers report durations, never timestamps; the cross-instance queue clock is labelled `realtime` |
| **Environment** | every telemetry record carries the worker's numpy and Python versions, CPU affinity, cgroup CPU limit, BLAS threads, precision, cap, coupling map and calibration file |
| **Resources** | identical CPU and memory limits for every worker; `OMP_NUM_THREADS=1` |
| **Builds** | `uv.lock` used with `--locked` inside Docker; pinned base images (`python:3.13.16-slim-trixie`, `uv:0.12.23`) |
| **Device data** | every calibration value tagged `published` or `assumed`; per-job `parameters_used`; strict provenance mode |
| **Benchmarks** | raw JSON with environment metadata, the plotting script and per-figure CSVs are all versioned in `benchmarks/` |

## Reporting checklist

- [ ] Report the QPUTwin version, the `uv.lock` hash and the Docker image digests.
- [ ] Name the calibration (`environment.worker.calibration`) and coupling map; prefer dated snapshots.
- [ ] List the `assumed_parameters` your results depend on, or run with `QPUTWIN_REQUIRE_PUBLISHED_CALIBRATION=1`.
- [ ] Report seeds (shot and routing), shots and, for trajectories, the trajectory count and `exact_sampling`.
- [ ] Report medians and percentiles over many jobs, with the host load, CPU and worker limits.
- [ ] Keep `OMP_NUM_THREADS=1` and access logging off for latency studies.
- [ ] Publish the `telemetry.jsonl` subset behind your figures.
