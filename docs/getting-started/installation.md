---
title: Installation
description: Install QPUTwin with Docker Compose or uv, verify the stack, and configure resources for the quantum digital twin workers.
---

# Installation

## Requirements

| Component | Version | Needed for |
|---|---|---|
| Docker Engine with Compose v2 | 24 or newer | the containerised stack (recommended) |
| [`uv`](https://docs.astral.sh/uv/) | 0.12 or newer | local development, tests, benchmarks |
| Python | 3.12 or 3.13 (`uv` installs it) | the library and the tests |
| Memory | 12 GiB per worker by default | statevector widths up to 28 qubits (29 at complex64) |
| OS | Linux (x86-64); macOS works with Docker Desktop | — |

No quantum SDK is required. QPUTwin does not depend on Qiskit, Cirq, pyQuil, Braket or pytket.

## Option A: Docker Compose (recommended)

```bash
git clone https://github.com/Cloudslab/QPUTwin.git qputwin && cd qputwin
docker compose up -d --build --wait
```

This builds two images (`qputwin/orchestrator`, which needs no numpy, and `qputwin/worker`) and starts eight services:

| Service | Image | Port | Role |
|---|---|---|---|
| `orchestrator` | `qputwin/orchestrator:0.1.0` | `8000` (host) | public API, validation, routing, telemetry, async queue |
| `worker-quantinuum-helios` | `qputwin/worker:0.1.0` | internal | Helios digital twin (98 qubits, all-to-all) |
| `worker-ionq-forte` | `qputwin/worker:0.1.0` | internal | Forte digital twin (36 qubits, all-to-all) |
| `worker-rigetti-ankaa3` | `qputwin/worker:0.1.0` | internal | Ankaa-3 digital twin (84 qubits, 7×12 lattice) |
| `worker-iqm-emerald` | `qputwin/worker:0.1.0` | internal | Emerald digital twin (54 qubits, 6×9 lattice) |
| `worker-ibm-heron` | `qputwin/worker:0.1.0` | internal | Heron r3 digital twin (156 qubits, heavy-hex, IBM's `ibm_boston` snapshot) |
| `worker-ibm-nighthawk` | `qputwin/worker:0.1.0` | internal | Nighthawk digital twin (120 qubits, square lattice, IBM's `ibm_miami` snapshot) |
| `worker-google-willow` | `qputwin/worker:0.1.0` | internal | Willow digital twin (105 qubits, square grid, Google's `willow_pink` snapshot) |

### Verify

```bash
curl -s localhost:8000/health          # {"status":"ok",...}
curl -s localhost:8000/ready           # job store + backends reachable
curl -s localhost:8000/v1/backends | python3 -m json.tool
```

Interactive OpenAPI documentation is served at <http://localhost:8000/docs>.

### Resources

Every worker gets the same CPU and memory limits, so latency comparisons reflect physics rather than resource contention. Set them in `.env`:

```dotenv
QPUTWIN_WORKER_CPUS=4      # CPU limit per worker container
QPUTWIN_WORKER_MEM=12g     # memory limit per worker container
QPUTWIN_PRECISION=complex128
```

The memory guard reads the same cgroup limit and rejects jobs that would not fit, so a worker never gets OOM-killed. See [Simulation engines](../guide/engines.md#memory-and-width-limits).

### Stop and remove

```bash
docker compose down              # stop, keep ./data (telemetry + job store)
docker compose down --rmi local  # also remove the images
```

## Option B: local Python environment (`uv`)

From a clone of the repository (see Option A):

```bash
uv sync                                    # Python env with every extra + dev tools
uv run pytest                              # 616 unit + in-process integration tests
```

Run the services without Docker, one terminal each:

```bash
QPUTWIN_VENDOR=ionq_forte QPUTWIN_PORT=8102 uv run python -m qputwin_worker
QPUTWIN_BACKEND_URL_IONQ_FORTE=http://localhost:8102 uv run python -m qputwin_orchestrator
```

Or use the physics directly, with no services:

```python
from qputwin_core.devices import DEVICES
from qputwin_core.simulators import ExecutionOptions, simulator_for

sim = simulator_for(DEVICES["ionq_forte"])
r = sim.execute(open("bell.qasm").read(), ExecutionOptions(shots=1000, seed=1))
print(r.counts)
```

## Optional dependency groups

| Group | Command | Contents |
|---|---|---|
| `dev` (default) | `uv sync` | pytest, httpx, all extras |
| `bench` | `uv run --group bench …` | matplotlib for the benchmark figures |

Package extras for custom images: `engine` (numpy, scipy), `worker` (engine + FastAPI + uvicorn) and `orchestrator` (FastAPI + uvicorn + httpx).

Next: [Quick start](quickstart.md).
