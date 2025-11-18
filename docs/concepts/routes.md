---
title: Routes
description: How Veilgate matches incoming requests to routes and forwards them to upstreams.
---

# Routes

Routes describe how Veilgate matches incoming HTTP requests and where they should be forwarded.

Each route can match on:

- **Host** – HTTP `Host` header.
- **Path** – path prefix or pattern.
- **Method** – HTTP method such as `GET`, `POST`.
- **Optional predicates** – headers or query parameters.

Matched routes are compiled into a **Route Runtime** which holds:

- upstream pool reference,
- middleware pipeline (auth, rate limiting, transforms),
- metadata (route ID, name, tags) used for metrics and observability.

See the configuration reference for the exact YAML structure of routes.


