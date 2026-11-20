---
title: Security
description: Secure a QPUTwin deployment — API keys, per-client job isolation, network isolation, TLS termination, secrets and input limits.
---

# Security

## API keys

```dotenv
QPUTWIN_API_KEYS=alice:9f3c1e7b2a4d6f80c1e2:9,bob:0b1c2d3e4f5a6b7c8d9e:3
# or mount a Docker secret and set QPUTWIN_API_KEYS_FILE=/run/secrets/qputwin_keys
```

Each entry is `name:key[:max_priority]`. Keys must be at least 16 characters; `max_priority` (0–9, default 9) caps the `priority` a client may request.

Once any key is configured:

* every `/v1/*` endpoint requires `Authorization: Bearer <key>` or `X-API-Key: <key>`, and returns `401 AuthenticationError` (with `WWW-Authenticate: Bearer`) otherwise;
* keys are compared in constant time;
* each client sees **only its own** asynchronous jobs. Someone else's job returns `404`, so job ids cannot be probed;
* the key name is the client identity for fair-share scheduling;
* `/health`, `/ready` and `/metrics` stay open for probes and scrapers. Restrict them at the network layer if needed.

## Network isolation

* Workers publish no ports; they are reachable only on the internal `qputwin-net` network.
* Only the orchestrator (`QPUTWIN_ORCHESTRATOR_PORTS`) and, in the `ha` profile, the gateway publish ports.
* Containers run as non-root users.

## TLS

QPUTwin serves plain HTTP inside the host. Terminate TLS at your edge proxy or load balancer (Caddy, Traefik, nginx, a cloud load balancer) in front of the orchestrator or the gateway.

## Input limits

| Limit | Default | Variable |
|---|---|---|
| QASM size | 1 MiB | `QPUTWIN_MAX_QASM_BYTES` |
| flattened operations | 5 000 000 | `QPUTWIN_MAX_OPS` |
| shots | 1 000 000 | `QPUTWIN_MAX_SHOTS` |
| queued async jobs | 10 000 | `QPUTWIN_ASYNC_MAX_QUEUED` |
| memory per job | container limit | memory guard (never OOM) |

The OpenQASM parser never calls `eval`, and `include` resolves only the embedded `qelib1.inc` and `stdgates.inc`; it never touches the filesystem.

## Not included

There is no built-in rate limiting and no TLS; use your edge proxy for both.
