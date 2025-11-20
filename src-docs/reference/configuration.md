---
title: Configuration reference
description: High-level reference for the veilgate.yaml configuration file.
---

# Configuration reference

Veilgate is configured using a YAML file, typically named `veilgate.yaml`.

High‑level sections include:

- `server` – listener addresses, timeouts, connection limits.
- `admin` – admin/metrics/dashboard listener and authentication.
- `logging` – log level, format and outputs.
- `metrics` – Prometheus metrics settings.
- `upstreams` – upstream pools and their endpoints.
- `routes` – route matchers, policies and upstream references.
- `security` – API keys, JWT issuers and related options.
- `rate_limit` – rate limit policies and defaults.
- `mcp` – MCP tool definitions and mappings (once configured).

See example configs under `examples/` and in release packages for concrete values.


