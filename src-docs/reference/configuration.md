---
title: Configuration reference
description: High-level reference for the veilgate.yaml configuration file.
---

# Configuration reference

Veilgate is configured using a YAML file, typically named `veilgate.yaml`.

High‑level sections include:

- `server` – listener addresses, timeouts, connection limits.
- `admin` – admin/metrics/dashboard listener.
- `logging` – log level, format and outputs.
- `metrics` – Prometheus metrics settings.
- `upstreams` – upstream pools and their endpoints.
- `routes` – route matchers, policies and upstream references.
- `security` – API keys, JWT issuers and related options.
- `rate_limit` – rate limit policies and defaults.
- `mcp` – MCP tool definitions and mappings (once configured).
- `admin_auth` – admin users, sessions and admin API rate‑limits.

See example configs under `examples/` and in release packages for concrete values.

### Logging – request & correlation IDs

Veilgate automatically ensures every incoming request carries a `request_id` for
consistent debugging:

- Incoming `X-Request-ID` headers are respected; if missing, the gateway
  generates a new opaque ID and returns it via the same header.
- `X-Correlation-ID` headers are copied through unchanged so upstream systems
  can stitch together multi-hop traces.
- Both values are stored in the request context and emitted with every structured
  log entry (`request_id`, `correlation_id` fields), making it easy to filter
  logs per call chain.

No extra configuration is required beyond enabling the default logging
middleware; customise formats and levels via the `logging` section as before.

### Metrics – auth & rate-limit counters

In addition to the core HTTP metrics, Veilgate now exports dedicated security
counters whenever `metrics` are enabled:

- `veilgate_auth_failures_total{route_id,reason}` counts API key and JWT
  failures. `reason` is either `api_key`, `jwt`, or `unknown`.
- `veilgate_ratelimit_rejections_total{route_id,scope}` counts requests rejected
  by rate limiting. `scope` reflects the evaluated limiter (`ip`, `route`,
  `api_key`, or `unknown` when not set).

Use these series to alert on unexpected auth errors or aggressive clients.

### `admin_auth` – admin users & dashboard login

The optional `admin_auth` section controls how admin users are stored and how
the dashboard `/dashboard` and admin API `/admin/*` are protected.

Example:

```yaml
admin_auth:
  store_path: ./data/admin-users.db
  seed_admin_user: admin
  seed_admin_password_hash: "<bcrypt-hash>"
  # or, to read plaintext from an env var and hash it at startup:
  # seed_admin_password_env: VEILGATE_ADMIN_BOOTSTRAP_PASSWORD
  rate_limit:
    enabled: true
    requests_per_second: 10
    burst: 20
```

Key fields:

- `store_path` – path to the embedded admin user database (bbolt file). If not
  set, a sensible default under the process working directory is used.
- `seed_admin_user` / `seed_admin_password_hash` – optional initial admin
  credentials; the password must be provided as a bcrypt hash.
- `seed_admin_password_env` – optional name of an environment variable holding
  the **plaintext** password to be hashed at startup (convenient for dev/test).
- `rate_limit` – basic rate limit settings for the admin API (same shape as
  `rate_limit.default` used for routes).


