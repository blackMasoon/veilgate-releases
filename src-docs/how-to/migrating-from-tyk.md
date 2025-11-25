---
title: Migrating from Tyk
description: Step-by-step guide for migrating API definitions from Tyk Gateway to Veilgate.
---

# Migrating from Tyk

This guide helps you migrate existing Tyk Gateway configurations to Veilgate,
covering the automated migration tool and manual configuration adjustments.

## Prerequisites

- Access to your Tyk API definition files (`apis/*.json`)
- Optional: CORS configuration files (`config/cors-*.json`)
- Go 1.21+ installed (for running the migration tool)
- Understanding of your current Tyk setup

## Quick migration

### Step 1: Run the migration tool

Veilgate includes an automated migration tool that converts Tyk API definitions
to Veilgate format:

```bash
# From the Veilgate repository root
go run ./scripts/tyk-to-veilgate/convert.go \
  -apis=/path/to/tyk/apis \
  -cors=/path/to/tyk/config \
  -output=./veilgate.yaml
```

The tool outputs warnings for features that require manual attention.

### Step 2: Review the generated configuration

Open the generated `veilgate.yaml` and verify:

- **Routes**: Ensure paths and methods match your expectations
- **Upstreams**: Verify backend URLs are correct
- **Authentication**: Configure secrets for API keys and JWT issuers
- **Rate limits**: Adjust values based on your traffic patterns

### Step 3: Add secrets

The migration tool does not copy secrets. Add them manually:

```yaml
security:
  api_keys:
    - id: mobile-app
      key: "your-secret-key-here"
      label: "Mobile Application"
  jwt_issuers:
    - id: main-idp
      algorithm: JWKS
      jwks_url: "https://auth.example.com/.well-known/jwks.json"
      jwks_cache_ttl_seconds: 300
```

### Step 4: Test the configuration

```bash
# Validate the configuration
./veilgate validate-config -config=./veilgate.yaml

# Start in development mode
./veilgate serve -config=./veilgate.yaml
```

## Feature mapping reference

### listen_path and strip_listen_path

**Tyk:**
```json
{
  "proxy": {
    "listen_path": "/ext/orders/",
    "strip_listen_path": true,
    "target_url": "http://orders:8080/api/v1"
  }
}
```

**Veilgate:**
```yaml
routes:
  - id: orders
    path: /ext/orders/*
    upstream_id: orders-upstream
    rewrite:
      strip_prefix: /ext/orders
      add_prefix: /api/v1

upstreams:
  - id: orders-upstream
    endpoints:
      - url: http://orders:8080
```

### CORS configuration

**Tyk (inline):**
```json
{
  "CORS": {
    "enable": true,
    "allowed_origins": ["https://app.example.com"],
    "allowed_methods": ["GET", "POST"],
    "allowed_headers": ["*"],
    "allow_credentials": true,
    "max_age": 24
  }
}
```

**Tyk (separate file `config/cors-orders.json`):**
```json
{
  "enable": true,
  "allowed_origins": ["https://app.example.com"]
}
```

**Veilgate:**
```yaml
routes:
  - id: orders
    cors:
      enable: true
      allowed_origins:
        - https://app.example.com
      allowed_methods:
        - GET
        - POST
      allowed_headers:
        - "*"
      allow_credentials: true
      max_age: 24
```

### Response headers

**Tyk:**
```json
{
  "global_response_headers": {
    "Strict-Transport-Security": "max-age=31536000",
    "X-Content-Type-Options": "nosniff"
  },
  "global_response_headers_remove": ["Server", "X-Powered-By"]
}
```

**Veilgate:**
```yaml
routes:
  - id: api
    response_headers:
      add:
        Strict-Transport-Security: "max-age=31536000"
        X-Content-Type-Options: nosniff
      remove:
        - Server
        - X-Powered-By
```

For global headers, use `server.response_headers`:

```yaml
server:
  response_headers:
    add:
      Strict-Transport-Security: "max-age=31536000"
    remove:
      - Server
```

### JWT authentication (JWKS)

**Tyk:**
```json
{
  "use_keyless_access": false,
  "jwt_signing_method": "RSA",
  "jwt_source": "https://auth.example.com/.well-known/jwks.json",
  "jwt_identity_base_field": "sub"
}
```

**Veilgate:**
```yaml
security:
  jwt_issuers:
    - id: main-idp
      algorithm: JWKS
      jwks_url: https://auth.example.com/.well-known/jwks.json
      jwks_cache_ttl_seconds: 300
      audience:
        - your-app

routes:
  - id: protected-api
    auth:
      jwt_issuer_id: main-idp
```

### API key authentication

**Tyk:**
```json
{
  "use_standard_auth": true,
  "auth": {
    "auth_header_name": "Api-key"
  }
}
```

**Veilgate:**
```yaml
security:
  api_keys_header_name: "Api-key"
  api_keys:
    - id: service-a
      key: "secret-key"

routes:
  - id: internal-api
    auth:
      api_key: true
```

### Rate limiting

**Tyk:**
```json
{
  "disable_rate_limit": false,
  "global_rate_limit": {
    "rate": 1000,
    "per": 60
  }
}
```

**Veilgate:**
```yaml
routes:
  - id: api
    rate_limit:
      enabled: true
      requests_per_second: 16.67  # 1000/60
      burst: 50
      scope: ip
```

### Proxy options

**Tyk:**
```json
{
  "proxy": {
    "preserve_host_header": true,
    "transport": {
      "ssl_insecure_skip_verify": true
    }
  }
}
```

**Veilgate:**
```yaml
routes:
  - id: legacy-api
    proxy:
      preserve_host_header: true
      tls:
        insecure_skip_verify: true
```

## Features not supported

The following Tyk features are not available in Veilgate:

| Feature | Alternative |
|---------|-------------|
| Go plugins | Move logic to external services |
| JSVM virtual endpoints | Use dedicated microservices |
| GraphQL schemas | Use a dedicated GraphQL gateway |
| Quota policies | Use external quota management |
| OAuth2 token generation | Use a dedicated identity provider |
| Custom middleware chains | Use Veilgate's built-in middleware |

## Dashboard migration

After migrating your API definitions, use the Veilgate dashboard to:

1. **View routes**: Navigate to Routes view to see all migrated routes
2. **Edit configurations**: Use the route editor for CORS, headers, and caching
3. **Manage API keys**: Generate and manage keys in the Policies view
4. **Monitor traffic**: Check statistics in the Overview and route details

Access the dashboard at `http://localhost:9090/dashboard` (admin port).

## Testing the migration

### Health checks

```bash
# Check gateway health
curl http://localhost:8080/healthz

# Check admin health
curl http://localhost:9090/healthz
```

### Verify routes

```bash
# List all routes via admin API
curl http://localhost:9090/admin/routes | jq

# Test a specific endpoint
curl -H "Api-key: your-key" http://localhost:8080/ext/orders/
```

### Compare responses

Test equivalent requests against both Tyk and Veilgate to verify behavior:

```bash
# Against Tyk
curl -v https://tyk-gateway/ext/orders/123

# Against Veilgate
curl -v http://localhost:8080/ext/orders/123
```

## Rollback plan

If issues arise, you can run Tyk and Veilgate in parallel:

1. Keep Tyk running on its original ports
2. Run Veilgate on different ports (e.g., 8081, 9091)
3. Use a load balancer to gradually shift traffic
4. Monitor error rates and latency
5. Complete migration once confident

## Getting help

- Review the [configuration reference](../reference/configuration.md)
- Check the [admin API documentation](../reference/admin-api.md)
- Open an issue on GitHub for specific migration challenges
