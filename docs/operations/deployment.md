---
title: Deployment
description: Deploy QPUTwin with Docker Compose — images, services, volumes, resource sizing, reproducible uv-locked builds and upgrades.
---

# Deployment

## The stack

```bash
docker compose up -d --build --wait
docker compose ps
```

| Item | Value |
|---|---|
| Compose project | `qputwin` |
| Images | `qputwin/orchestrator:0.1.0` (no numpy) and `qputwin/worker:0.1.0` |
| Base images | `python:3.13.16-slim-trixie`, `ghcr.io/astral-sh/uv:0.12.23` (pinned) |
| Network | `qputwin-net` (internal bridge); only the orchestrator, and optionally the gateway, publish ports |
| Volumes | `./data` → `/app/data` (telemetry dataset, job store); `./config` → `/app/config` (read-only) |
| User | non-root (`qputwin`, uid 10001 in the images; the orchestrator runs as `QPUTWIN_UID:QPUTWIN_GID` to own `./data`) |
| Health checks | `/health` on every service |
| License | Apache-2.0 (`LICENSE` and `NOTICE` ship in both images) |

## Reproducible builds

Each image is built in two cached layers:

1. `uv sync --locked --no-install-project` installs the third-party dependencies from `uv.lock`.
2. The project itself is installed non-editable.

Editing code therefore never reinstalls numpy or scipy, and a fresh build reproduces the exact locked versions.

## Sizing

| Workload | Per-worker memory | Notes |
|---|---|---|
| ideal ≤ 20 qubits | 1 GiB | any machine |
| ideal 28 qubits | 12 GiB (default) | `complex64` reaches 29 |
| exact noise ≤ 14 qubits | 12 GiB | density matrix |
| wide noisy circuits | as ideal | trajectory engine; time grows with the trajectory count |

CPU: BLAS threading is off (`OMP_NUM_THREADS=1`); the fused engine uses the worker's CPU quota (`QPUTWIN_WORKER_CPUS`, default 4). On one host, adding worker replicas scales 20-qubit jobs linearly (3.10× with three replicas), while small, cache-resident jobs are limited by the orchestrator's dispatch path (1.41× with three) and need orchestrator replicas instead (see the [scaling benchmark](../research/benchmarks.md#horizontal-scaling)).

## Data

| File | Contents | Safe to delete? |
|---|---|---|
| `data/telemetry.jsonl` | one JSON line per job (the research dataset) | yes, but this loses the dataset |
| `data/jobs.sqlite3` (+ `-wal`, `-shm`) | asynchronous jobs | only when no jobs are queued or running |

!!! warning "Filesystem for the job store"
    SQLite in WAL mode needs a local filesystem with working shared-memory `mmap`. Native Linux filesystems (ext4, xfs, btrfs) are recommended. Network filesystems (NFS, SMB) are not supported.

## Upgrades

```bash
git pull
docker compose up -d --build --wait     # rolling: workers first, then the orchestrator (depends_on)
```

Running asynchronous jobs survive an orchestrator restart. A graceful stop releases their leases and the restarted instance resumes them.

## Without Docker

Run the services with `uv run python -m qputwin_worker` / `qputwin_orchestrator` under systemd or another supervisor, setting the same environment variables. See [Installation](../getting-started/installation.md#option-b-local-python-environment-uv).
