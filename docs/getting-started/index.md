---
title: Getting started with QPUTwin
description: Install QPUTwin, run your first OpenQASM job on a quantum digital twin, and compare four quantum processors in a short tutorial.
---

# Getting started

QPUTwin runs as a small Docker Compose stack: one **orchestrator** that accepts OpenQASM jobs over HTTP, and four **worker** containers, one digital twin per quantum processor. You can also use the physics core directly as a Python library.

| Step | Page | Time |
|---|---|---|
| 1. Install with Docker or `uv` | [Installation](installation.md) | 5 min |
| 2. Submit ideal, noisy, routed and asynchronous jobs | [Quick start](quickstart.md) | 10 min |
| 3. Run one circuit on four of the QPUs and interpret the results | [Tutorial](tutorial.md) | 20 min |

!!! tip "Which interface should I use?"
    * **HTTP API (Docker stack):** use this for benchmarking, multi-user access, asynchronous jobs and anything that measures latency. It is the interface the research telemetry describes.
    * **Python library (`qputwin_core`):** use this for notebooks, tests and parameter sweeps where you want the physics without HTTP. See [Python library](../guide/python-library.md).
