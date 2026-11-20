---
title: FAQ
description: Frequently asked questions about QPUTwin, the open-source quantum digital twin and QPU emulator — SDKs, noise realism, vendor data, qubit limits, routing and latency benchmarking.
---

# FAQ

??? question "What is QPUTwin?"
    QPUTwin is an open-source quantum digital twin and QPU emulator. It reproduces trapped-ion (Quantinuum Helios, IonQ Forte) and superconducting (Rigetti Ankaa-3, IQM Emerald) quantum processors behind a single OpenQASM 2.0/3.x REST API: their native gates, qubit connectivity, calibrated noise and gate timing.

??? question "Does QPUTwin need Qiskit, Cirq or another quantum SDK?"
    No. The OpenQASM parser, compiler, routing and the statevector, density-matrix and trajectory simulators are implemented from scratch in numpy and scipy. Any tool that emits OpenQASM can submit circuits.

??? question "How realistic is the noise model?"
    Every physical gate's error channel is solved to reproduce the device's published average gate fidelity **exactly**, together with T1/T2 relaxation during gates and idle windows, coherent error, crosstalk, leakage, quasi-static detuning, reset and readout errors. Each result lists which calibration values were published and which were assumed. See [Noise models](guide/noise.md).

??? question "Can I use real calibration data from a vendor?"
    Yes. Export Amazon Braket device properties and convert them with `scripts/import_braket_device.py`, or drop any coupling map or calibration JSON into `config/devices/`. No code change is needed. See [Calibration data](guide/calibration.md).

??? question "How many qubits can QPUTwin simulate?"
    In a 12 GiB worker:

    * ideal and noisy-trajectory runs reach 28 qubits (29 at complex64);
    * exact density-matrix runs reach 14.

    Only qubits a circuit touches are simulated, so small circuits on the 98-qubit Helios twin are cheap. See [Simulation engines](guide/engines.md).

??? question "What happens when a circuit violates a chip's connectivity?"
    By default it is rejected with `TopologicalCompilationError`, quoting the offending statement, line and qubit pair. With `options.routing = {"method": "sabre"}` it is routed instead, and the SWAP overhead is reported. See [Qubit routing](guide/routing.md).

??? question "Can QPUTwin be used to benchmark quantum-cloud latency?"
    Yes. Every response carries telemetry whose components add up exactly to the round trip, and every job is recorded in a JSONL dataset. See [Telemetry](guide/telemetry.md) and the [benchmarks](research/benchmarks.md).

??? question "Is QPUTwin production-ready?"
    It provides:

    * a durable asynchronous queue, with leases and crash recovery;
    * priority and fair-share scheduling;
    * API keys with per-client isolation;
    * worker and orchestrator replicas behind an HA gateway;
    * Prometheus metrics and readiness probes.

    Its limits are listed on [Limitations](research/limitations.md).

??? question "Can I trust the absolute numbers?"
    Trends and ratios, yes. Absolute noise and timing values depend on the calibration; check `assumed_parameters`, or load a dated vendor snapshot and enable strict provenance.
