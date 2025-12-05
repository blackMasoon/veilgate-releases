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
- `license` – enterprise license token and options.
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

Set `logging.access.enabled: false` to completely remove the access-log
middleware from the hot path in high-throughput environments (fewer allocations,
no per-request JSON log). When disabled, request/correlation IDs are not
attached to the context, so only enable this mode if you prefer raw performance
over diagnostic richness. Customise formats and levels via the rest of the
`logging` section as before.

### Metrics – auth & rate-limit counters

In addition to the core HTTP metrics, Veilgate now exports dedicated security
counters whenever `metrics` are enabled:

- `veilgate_auth_failures_total{route_id,reason}` counts API key and JWT
  failures. `reason` is either `api_key`, `jwt`, or `unknown`.
- `veilgate_ratelimit_rejections_total{route_id,scope}` counts requests rejected
  by rate limiting. `scope` reflects the evaluated limiter (`ip`, `route`,
  `api_key`, or `unknown` when not set).

Use these series to alert on unexpected auth errors or aggressive clients. The
associated `/admin/stats/*` snapshots can be disabled via
`metrics.detailed_stats: false`, which removes per-request lock contention for
the in-memory counters while leaving Prometheus metrics untouched. This is
recommended for latency-sensitive production or benchmark profiles.

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

### `security` – API keys & JWT issuers

The `security` section configures shared authentication primitives that routes
can reference later on:

- `api_keys` – static keys defined up front (id, key, optional label/created_at).
- `api_keys_header_name` – global header that carries API keys on every route.
  Defaults to `X-API-Key`, but you can switch to Tyks `Api-key` or any other
  canonical header without recompiling the gateway.
- `jwt_issuers` – trusted JWT providers. Each issuer advertises:
  - `id` – unique reference used by routes (`routes[].auth.jwt_issuer_id`).
  - `algorithm` – `HS256`, `RS256` (local PEM), or `JWKS` for remote key sets.
  - Depending on the algorithm, additional fields such as `hs256_secret`,
    `rsa_public_key_path`, `jwks_url`, and `jwks_cache_ttl_seconds` are required.
  - Optional `issuer` and `audience[]` constraints are enforced during token
    validation.

Routes can still override the header name on a per-route basis via
`routes[].auth.api_key_header_name`. If left empty, the global header name
applies.

Example:

```yaml
security:
  api_keys_header_name: "Api-key"
  api_keys:
    - id: m2m-service
      key: demo-secret-key
      label: "Internal service"
  jwt_issuers:
    - id: accounts
      issuer: "https://id.example.com/"
      audience: ["truckmanager"]
      algorithm: JWKS
      jwks_url: "https://id.example.com/.well-known/jwks.json"
      jwks_cache_ttl_seconds: 300
```

In the example above every route that enables `auth.api_key: true` expects the
client to send `Api-key: demo-secret-key`. Any route that references the
`accounts` issuer validates RS256 tokens by discovering signing keys from the
JWKS URL and caching them for five minutes.

### `routes[].cache` – HTTP response caching

Veilgate provides a simple in-memory cache for idempotent requests (GET/HEAD)
to reduce upstream load and improve latency. Configure caching per route:

```yaml
routes:
  - id: cached-api
    path: /api/v1/products
    method: GET
    upstream_id: products-svc
    cache:
      enable: true
      ttl_seconds: 300
      vary_by_headers:
        - Accept-Language
        - Accept-Encoding
      cache_all_safe_requests: false
      honor_upstream_cache_control: true
```

Key fields:

- `enable` – must be `true` to activate caching for the route.
- `ttl_seconds` – how long cached responses remain valid (default is route TTL).
- `vary_by_headers` – list of request headers to include in the cache key
  (e.g., `Accept-Language` produces separate cache entries per language).
- `cache_all_safe_requests` – when `true`, also caches OPTIONS requests.
- `honor_upstream_cache_control` – when `true`, respects `Cache-Control: no-store`,
  `Cache-Control: private`, and `max-age` directives from upstream responses.

Cache metrics are exposed via Prometheus:
- `veilgate_cache_events_total{route_id,event}` where `event` is `hit`, `miss`,
  or `store`.

### `routes[].proxy` – Proxy and transport options

Each route can customize how requests are proxied to upstreams:

```yaml
routes:
  - id: legacy-api
    path: /legacy/*
    method: "*"
    upstream_id: legacy-backend
    proxy:
      preserve_host_header: true
      tls:
        insecure_skip_verify: true
      timeouts:
        dial_seconds: 5
        tls_handshake_seconds: 5
        response_header_seconds: 30
        expect_continue_seconds: 1
```

Key fields:

- `preserve_host_header` – when `true`, the original `Host` header from the
  client request is passed to the upstream (instead of the upstream's hostname).
  Useful for virtual-host routing on legacy backends.
- `tls.insecure_skip_verify` – skip TLS certificate verification for upstream
  connections. **Use only for development or trusted internal services.**
- `timeouts.dial_seconds` – TCP dial timeout to upstream.
- `timeouts.tls_handshake_seconds` – TLS handshake timeout.
- `timeouts.response_header_seconds` – time to wait for response headers.
- `timeouts.expect_continue_seconds` – time to wait for 100 Continue response.

The same `proxy` block can be specified at the **upstream level** (`upstreams[].proxy`)
to set defaults for all routes using that upstream. Route-level settings override
upstream-level settings.

### `server.http` – Global HTTP client tuning

Global transport settings for all upstream connections:

```yaml
server:
  http:
    max_idle_conns: 512
    max_idle_conns_per_host: 128
    idle_conn_timeout_seconds: 90
    dial_timeout_seconds: 30
    tls_handshake_timeout_seconds: 10
    response_header_timeout_seconds: 30
    expect_continue_timeout_seconds: 1
```

These defaults apply to all upstreams unless overridden by `upstreams[].proxy`
or `routes[].proxy`.

### `routes[].rewrite` – Path rewriting

Control how request paths are transformed before proxying:

```yaml
routes:
  - id: api-v2
    path: /ext/api/v2/*
    method: GET
    upstream_id: api-backend
    rewrite:
      strip_prefix: /ext
      add_prefix: /internal
```

- `strip_prefix` – removes the specified prefix from the request path before
  forwarding. The prefix must match the beginning of the configured route path.
- `add_prefix` – prepends the specified prefix to the (possibly stripped) path.

Example transformation: `/ext/api/v2/users` → `/internal/api/v2/users`

### `routes[].response_headers` – Response header manipulation

Add or remove headers from responses:

```yaml
routes:
  - id: secure-api
    path: /api/*
    method: "*"
    upstream_id: backend
    response_headers:
      add:
        Strict-Transport-Security: "max-age=31536000; includeSubDomains"
        X-Content-Type-Options: nosniff
        X-Frame-Options: DENY
      remove:
        - Server
        - X-Powered-By
```

Global response headers can be configured under `server.response_headers` and
are applied before route-specific overrides.

### `metrics.sink` – Stats export (Pump alternative)

Export periodic stats snapshots to external stores. Currently supports JSONL
file output as a lightweight alternative to Tyk Pump:

```yaml
metrics:
  enabled: true
  sink:
    jsonl:
      path: /var/log/veilgate/stats.jsonl
      interval_seconds: 60
```

Each snapshot includes:
- `timestamp` – when the snapshot was taken.
- `route_stats` – per-route request counts and latency.
- `api_key_stats` – per-API-key usage statistics.
- `auth_failures` – authentication failure counts by route and reason.
- `rate_limit_rejections` – rate limit rejection counts by route and scope.

The JSONL format (one JSON object per line) is suitable for log aggregation
tools like Loki, Elasticsearch, or custom processing pipelines.

### `routes[].ip_filter` – IP-based access control

Configure CIDR-based allow/deny lists for fine-grained IP filtering per route:

```yaml
routes:
  - id: admin-api
    path: /admin/*
    method: "*"
    upstream_id: admin-backend
    ip_filter:
      allow_cidrs:
        - 10.0.0.0/8
        - 192.168.1.0/24
        - 172.16.0.0/12
      deny_cidrs:
        - 0.0.0.0/0
```

Key fields:

- `allow_cidrs` – list of CIDR ranges that are allowed to access this route.
  If specified, only IPs matching these ranges can access the route.
- `deny_cidrs` – list of CIDR ranges that are explicitly denied. Deny rules
  are evaluated after allow rules.

**Evaluation order:**

1. If `allow_cidrs` is non-empty, the client IP must match at least one allowed CIDR.
2. If `deny_cidrs` is non-empty and the client IP matches any denied CIDR, access is denied.
3. If neither list is configured, all IPs are allowed.

The middleware extracts the client IP from:
1. The first IP in the `X-Forwarded-For` header (if present)
2. The remote address from the connection

### `routes[].cors` – CORS configuration

Enable Cross-Origin Resource Sharing (CORS) for browser-based API access:

```yaml
routes:
  - id: public-api
    path: /api/v1/*
    method: "*"
    upstream_id: api-backend
    cors:
      enable: true
      allowed_origins:
        - https://app.example.com
        - https://staging.example.com
      allowed_methods:
        - GET
        - POST
        - PUT
        - DELETE
        - OPTIONS
      allowed_headers:
        - Content-Type
        - Authorization
        - X-Requested-With
      allow_credentials: true
      max_age: 24
```

Key fields:

- `enable` – must be `true` to activate CORS handling for the route.
- `allowed_origins` – list of origins that are allowed to make cross-origin requests.
  Use `*` for wildcard (not recommended with credentials).
- `allowed_methods` – HTTP methods permitted in cross-origin requests.
- `allowed_headers` – request headers that clients may include in cross-origin requests.
- `allow_credentials` – whether to include credentials (cookies, auth headers) in requests.
  Note: `allow_credentials: true` is incompatible with wildcard origins.
- `max_age` – how long (in hours) browsers should cache preflight responses.

**Preflight handling:**

When CORS is enabled, Veilgate automatically handles `OPTIONS` preflight requests
by responding with appropriate `Access-Control-*` headers without forwarding to
the upstream. This reduces upstream load and improves response times.

### Complete route configuration example

Here's a complete example showing all available route options:

```yaml
routes:
  - id: orders-api-v2
    host: api.example.com
    path: /ext/orders/v2/*
    method: "*"
    upstream_id: orders-backend
    
    # Path rewriting
    rewrite:
      strip_prefix: /ext/orders/v2
      add_prefix: /api/orders
    
    # CORS for browser access
    cors:
      enable: true
      allowed_origins:
        - https://app.example.com
      allowed_methods:
        - GET
        - POST
        - PUT
        - DELETE
      allowed_headers:
        - "*"
      allow_credentials: false
      max_age: 24
    
    # Security headers
    response_headers:
      add:
        Strict-Transport-Security: "max-age=31536000; includeSubDomains"
        X-Content-Type-Options: nosniff
        X-Frame-Options: DENY
        Referrer-Policy: strict-origin-when-cross-origin
      remove:
        - Server
        - X-Powered-By
    
    # Response caching
    cache:
      enable: true
      ttl_seconds: 300
      vary_by_headers:
        - Accept-Language
      honor_upstream_cache_control: true
    
    # Proxy options
    proxy:
      preserve_host_header: false
      tls:
        insecure_skip_verify: false
      timeouts:
        dial_seconds: 5
        response_header_seconds: 30
    
    # IP filtering
    ip_filter:
      allow_cidrs:
        - 10.0.0.0/8
    
    # Authentication
    auth:
      api_key: true
      jwt_issuer_id: main-idp
    
    # Rate limiting
    rate_limit:
      enabled: true
      requests_per_second: 100
      burst: 200
      scope: api_key
```

### `license` – enterprise licensing

Veilgate enforces basic demo limits when no valid license is provided:

- Upstreams: max 1
- Routes: max 3

To enable full features, paste the issued license token:

```yaml
license:
  key: "VEIL-1.<base64url-payload>.<base64url-signature>"
  # optional instance binding and future online checks
  instance_id: "prod-gw-eu-1"
  server_url: "https://licenses.example.com"  # optional
```

Licenses are Ed25519-signed tokens that encode entitlements such as
`max_upstreams` and `max_routes`, optional expiry, and optional bindings
(`instance_id`, `domains[]`).

