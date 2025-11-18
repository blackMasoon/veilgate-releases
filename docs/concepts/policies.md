---
title: Policies: auth & rate limiting
description: Built-in security and rate limiting policies and how they attach to routes.
---

# Policies: auth & rate limiting

Veilgate ships with built‑in policies for authentication and rate limiting.

## Authentication

Supported mechanisms:

- **API keys** – static keys configured in `security.api_keys`.
- **JWT** – HS256/RS256 tokens with configurable issuer, audience and claim constraints.

Policies can be attached to routes to enforce:

- specific key sets,
- specific JWT issuers/audiences or claims.

## Rate limiting

Rate limiting is implemented as an efficient in‑memory token bucket.

Limits can be configured per:

- API key (consumer),
- client IP,
- route.

Policy configuration lives in the `rate_limit` section and is attached to routes via references.


