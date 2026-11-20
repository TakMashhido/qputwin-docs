---
title: Python library
description: Use QPUTwin's physics core (qputwin_core) directly from Python — simulators, execution options, devices, calibration overrides, noise and routing without HTTP.
---

# Python library

The physics is a regular Python package, `qputwin_core`. Install it with `uv sync` and use it in notebooks, tests and parameter sweeps.

## Run a circuit

```python
from qputwin_core.devices import DEVICES
from qputwin_core.simulators import ExecutionOptions, simulator_for

GHZ = """OPENQASM 2.0; include "qelib1.inc";
qreg q[3]; creg c[3]; h q[0]; cx q[0],q[1]; cx q[1],q[2]; measure q -> c;"""

sim = simulator_for(DEVICES["iqm_emerald"])          # SuperconductingSimulator
r = sim.execute(GHZ, ExecutionOptions(shots=4000, seed=1))
print(r.counts)                                      # {'000': 2015, '111': 1985}
print(r.compilation["native_gate_histogram"])        # {'cz': 2, 'prx': 5, 'rz': 5}
```

## `ExecutionOptions`

| Field | Default | Same as the HTTP option |
|---|---|---|
| `shots`, `seed` | 1024, None | `shots`, `seed` |
| `compilation` | `"native"` | `options.compilation` |
| `return_statevector`, `return_density_matrix`, `memory` | False | same |
| `precision` | None | `options.precision` |
| `routing` | None | a dict like `{"method": "sabre"}` or `RoutingOptions` |
| `noise` | None | a dict like `{"model": "device"}` or `NoiseOptions` |
| `engine` | `"auto"` | `options.engine` |

`ExecutionResult` has `counts`, `memory`, `statevector`, `statevector_qubits`, `density_matrix`, `noise`, `compilation`, `workload`, `seed` and `cregs`; `.to_dict()` gives the worker's JSON shape.

## Noise and routing

```python
r = sim.execute(GHZ, ExecutionOptions(shots=20_000, seed=2,
                                      noise={"model": "device", "scale": 2.0},
                                      routing={"method": "sabre", "bridge": True}))
print(r.noise["engine"], r.noise["estimated_success_probability"], r.noise["assumed_parameters"])
# density_matrix 0.78 ['gates.cz.duration_ns', 'gates.prx.duration_ns', 'readout.fidelity']
```

## Timing spans

```python
from qputwin_core.timing import SpanTimer

timer = SpanTimer()
sim.execute(GHZ, ExecutionOptions(shots=1024), timer)
print(timer.as_ms())    # {'parse': …, 'compile': …, 'simulation': …, 'sampling': …}
```

## Devices, overrides and custom topologies

```python
from qputwin_core.devices import DEVICES, apply_overrides
from qputwin_core.topology import CouplingMap
from qputwin_core.calibration import load_calibration_file

spec = DEVICES["rigetti_ankaa_3"]
line = CouplingMap(n_qubits=5, kind="graph", edges=((0, 1), (1, 2), (2, 3), (3, 4)))
tiny = spec.with_overrides(coupling=line)                         # a 5-qubit line "Ankaa"
snap = spec.with_overrides(calibration=load_calibration_file("config/devices/ankaa3_snapshot.json"))
same = apply_overrides(spec, {"QPUTWIN_COUPLING_MAP": "config/devices/example_rigetti_ankaa_3_defects.json"})
```

## Lower-level modules

| Module | Key API |
|---|---|
| `qputwin_core.qasm` | `load_circuit(source, primitives=…)` → `Circuit` (pure Python) |
| `qputwin_core.synthesis` | `NativeCompiler(spec).compile(circuit, mode)` → `NativeCircuit` |
| `qputwin_core.routing` | `SabreRouter(coupling, RoutingOptions(...)).run(circuit)`, `cancel_inverse_pairs(ops)` |
| `qputwin_core.scheduling` | `asap_schedule(native.ops, calibration)`, `duration_report(...)` |
| `qputwin_core.noise` | `NoiseModel`, `relaxation_kraus`, `depolarizing_parameter`, `build_noisy_program` |
| `qputwin_core.engine` / `density` / `trajectory` | `StatevectorEngine`, `DensityMatrixEngine`, `run`, `run_density`, `run_trajectories` |
| `qputwin_core.gates` | closed-form matrices: `G.matrix(name, params)`, `GATE_TABLE` |
| `qputwin_core.errors` | `QPUTwinError` hierarchy with `http_status`, `type`, `details` |
