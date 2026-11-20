---
title: Reference
description: Complete QPUTwin reference — HTTP API endpoints, every request parameter, response schema, error types, configuration variables and the calibration file format.
---

# Reference

| Page | Contents |
|---|---|
| [HTTP API](api.md) | every orchestrator and worker endpoint, with methods, parameters and examples |
| [Request parameters](parameters.md) | every field of `JobRequest`, `options`, `options.routing` and `options.noise`: type, default, range |
| [Response schema](responses.md) | `result`, `compilation`, `workload`, `noise` and `telemetry` field by field |
| [Errors](errors.md) | all 19 error types, their HTTP status codes and `details` |
| [Configuration](configuration.md) | all environment variables, `backends.toml` and Compose settings |
| [Calibration file format](calibration-format.md) | the JSON schema for calibrations and coupling maps |

Interactive OpenAPI documentation is also served by the running orchestrator at <http://localhost:8000/docs> (Swagger UI) and `/openapi.json`.
