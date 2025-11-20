---
title: Admin API & dashboard
description: How the admin server, JSON API and dashboard work together.
---

# Admin API & dashboard

Veilgate exposes a separate **admin server** for operators. It typically listens on a different port
than user traffic and exposes:

- health endpoints (`/healthz`, `/readyz`),
- Prometheus metrics (`/metrics`),
- JSON admin endpoints under `/admin/...`,
- built‑in dashboard under `/dashboard`.

The dashboard is a React SPA that consumes the admin JSON API to:

- display routes and their policies,
- display upstream pools and health,
- display MCP tools and their mappings (once configured),
- display summary statistics,
- **manage configuration in place** (when admin write access is enabled).

## Dashboard management capabilities (CRUD)

When the admin API is exposed with write access, the dashboard can:

- **Routes**
  - List all effective routes (as before).
  - **Create** new routes (`POST /admin/routes`) by filling in ID, method, host, path and upstream.
  - **Edit** existing routes (`PUT /admin/routes/{id}`) – core routing fields can be adjusted at runtime.
  - **Delete** routes (`DELETE /admin/routes/{id}`) – removed routes stop matching immediately.
  - Advanced auth and rate‑limit options are still primarily configured via YAML, but are validated and enforced on the updated runtime after each change.

- **Upstreams**
  - List upstream pools and endpoints.
  - **Create** new upstreams (`POST /admin/upstreams`) by specifying an ID and one or more endpoint URLs.
  - **Edit** upstream endpoints (`PUT /admin/upstreams/{id}`) – change or add/remove backend URLs.
  - **Delete** upstreams (`DELETE /admin/upstreams/{id}`); validation prevents removing upstreams still referenced by routes.

- **API keys**
  - View API keys defined in configuration (`security.api_keys`) as part of the policies view.
  - **Generate** new API keys via the admin API (`POST /admin/security/api-keys`); the dashboard:
    - shows the generated key **only once** so it can be copied,
    - updates the running configuration and key store immediately.
  - **Activate/deactivate** keys (`PUT /admin/security/api-keys/{id}`) by toggling their `active` status without removing them from history.
  - **Delete** keys (`DELETE /admin/security/api-keys/{id}`) so they can no longer be used.

All write operations go through the same safe flow:

1. The admin server applies the change to an in‑memory copy of the configuration.
2. The new config is validated against the same rules as the file‑based config.
3. Runtime components (upstream registry, auth, rate limiting, router) are rebuilt.
4. If everything succeeds, the new runtime is atomically swapped in; otherwise the dashboard shows a validation error and the previous config stays active.

See the [Admin API reference](../reference/admin-api.md) for endpoint‑level details and the write operations exposed under `/admin/routes`, `/admin/upstreams` and `/admin/security/api-keys`.


