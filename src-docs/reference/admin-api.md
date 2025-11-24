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
- `POST /admin/auth/login` – authenticate an admin user and establish a session (HTTP-only cookie).
- `POST /admin/auth/logout` – terminate the current admin session.
- `GET /admin/routes` – effective routes and their configuration.
- `POST /admin/routes` – create a new route and apply it at runtime.
- `PUT /admin/routes/{id}` – update an existing route.
- `DELETE /admin/routes/{id}` – delete an existing route.
- `GET /admin/upstreams` – upstream pools and health.
- `GET /admin/policies` – auth and rate‑limit policies.
- `GET /admin/mcp/tools` – MCP tools and mappings (once configured).
- `GET /admin/summary` – basic gateway summary (total routes and upstreams).
- `GET /admin/stats/routes` – basic usage statistics per route (requests, errors, latency).
- `GET /admin/stats/api-keys` – basic usage statistics per API key.
- `GET /admin/stats/security` – aggregated auth failures and rate-limit rejections per route.

The full, structured definition of the admin API is generated as an OpenAPI document
and published in the external documentation site.

Most `/admin/*` endpoints (including the dashboard under `/dashboard`) require a
valid admin session and are **rate-limited per client IP** to protect against abuse.
Errors are returned as JSON objects of the form:

```json
{ "error": "human readable message" }
```


