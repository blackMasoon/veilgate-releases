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

The dashboard is a React SPA that consumes the admin JSON API to display:

- routes and their policies,
- upstream pools and health,
- MCP tools and their mappings (once configured),
- summary statistics.

See the [Admin API reference](../reference/admin-api.md) for endpoint‑level details.


