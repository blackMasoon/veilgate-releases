---
title: Admin API
description: Reference documentation for the Veilgate admin HTTP API.
---

# Admin API

The admin server exposes a JSON API for operators under a dedicated listener.

Endpoints include:

- `GET /healthz` – basic liveness probe.
- `GET /readyz` – readiness probe.
- `GET /metrics` – Prometheus metrics.
- `GET /admin/routes` – effective routes and their configuration.
- `GET /admin/upstreams` – upstream pools and health.
- `GET /admin/policies` – auth and rate‑limit policies.
- `GET /admin/mcp/tools` – MCP tools and mappings (once configured).
- `GET /admin/summary` – basic gateway summary (total routes and upstreams).

The full, structured definition of the admin API is generated as an OpenAPI document
and published in the external documentation site.


