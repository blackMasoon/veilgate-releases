---
title: Upstreams
description: How Veilgate manages upstream pools, load balancing and health.
---

# Upstreams

Upstreams represent backend services that Veilgate proxies traffic to.

Key ideas:

- **Upstream pools** – named sets of endpoints (hosts) with load‑balancing strategy.
- **Health** – each endpoint can be marked as up, degraded or down based on active/passive checks.
- **Connection management** – Veilgate uses tuned HTTP transports for connection pooling and timeouts.

Routes reference upstream pools by ID, so you can:

- reuse upstreams across many routes,
- change upstream details without touching route definitions.


