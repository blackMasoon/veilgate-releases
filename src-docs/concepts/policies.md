---
title: Policies: auth & rate limiting
description: Built-in security and rate limiting policies and how they attach to routes.
---

# Policies: auth & rate limiting

Veilgate ships with built‑in policies for authentication and rate limiting.

## Authentication

Veilgate ships with two first-class authentication mechanisms that mirror the
Tyk setups we are replacing:

- **API keys** – static keys configured in `security.api_keys`. A single,
  configurable header carries keys across all routes
  (`security.api_keys_header_name`, default `X-API-Key`). When a particular API
  needs to honor a legacy header (`Api-key`, `X-Customer-Key`, etc.), override
  it per route via `routes[].auth.api_key_header_name`. The dashboard now
  surfaces both the global default and per-route overrides so operators can
  confirm exactly what the gateway expects. Keys created via the admin API can
  optionally carry a **per-key policy**: a rate limit bucket (RPS/burst) and
  static claims. Claims are forwarded to upstreams as `X-VG-Claim-<key>: <value>`
  headers to mimic Tyk’s key-scoped metadata.
- **JWT** – tokens issued by configurable providers. Veilgate supports HS256,
  RS256 with a local PEM, and RS256 via JWKS discovery. JWKS issuers declare a
  `jwks_url` and a positive `jwks_cache_ttl_seconds`; the gateway fetches and
  caches keys, selecting the right one via the token's `kid`. Optional `issuer`
  and `audience[]` claims are validated for every token.

Policies can be attached to routes to enforce:

- specific key sets or header names,
- specific JWT issuers/audiences or downstream claim constraints.

## Rate limiting

Rate limiting is implemented as an efficient in‑memory token bucket.

Limits can be configured per:

- API key (consumer),
- client IP,
- route.

Policy configuration lives in the `rate_limit` section and is attached to routes via references.

