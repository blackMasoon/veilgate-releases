---
title: Performance
description: Veilgate performance characteristics and benchmarking guide.
---

# Performance

Veilgate is designed to be a lightweight, high-performance API gateway with minimal
overhead compared to direct backend access.

## Performance goals

Veilgate targets the following performance characteristics:

| Metric | Target | Notes |
|--------|--------|-------|
| p95 latency overhead | ≤ 1ms | Pass-through proxy, small responses |
| Throughput | 10-20k RPS | 2 vCPU, simple routes without TLS |
| Memory footprint | < 50MB | Base memory with typical configuration |
| Startup time | < 1 second | Cold start to ready |

These targets assume:
- Single gateway instance
- No TLS termination at the gateway
- Small response bodies (1-4 KB)
- In-memory rate limiting and caching

## Running benchmarks

Veilgate includes a benchmark suite in `internal/perf/`. Run benchmarks with:

```bash
# Quick benchmark run
go test -bench=. ./internal/perf/...

# Extended run with more iterations
go test -bench=. -benchtime=5s ./internal/perf/...

# With memory allocation stats
go test -bench=. -benchmem ./internal/perf/...
```

### Available benchmarks

| Benchmark | Description |
|-----------|-------------|
| `BenchmarkGatewayRoundTrip` | Basic proxy pass-through |
| `BenchmarkGatewayWithCORS` | CORS middleware overhead |
| `BenchmarkGatewayWithRateLimit` | Rate limiting middleware overhead |
| `BenchmarkGatewayWithCache` | Response caching (cache hit) |
| `BenchmarkGatewayWithRewrite` | Path rewriting overhead |
| `BenchmarkGatewayWithResponseHeaders` | Response header manipulation |
| `BenchmarkGatewayAllFeatures` | All middleware combined |
| `BenchmarkGatewayLargeBody` | Large request/response handling |
| `BenchmarkGatewayManyRoutes` | Routing with 100 routes |

### Interpreting results

Example output:

```
BenchmarkGatewayRoundTrip-4      50000     28456 ns/op    1024 B/op    12 allocs/op
```

- `50000` – number of iterations
- `28456 ns/op` – nanoseconds per operation (~28 μs)
- `1024 B/op` – bytes allocated per operation
- `12 allocs/op` – heap allocations per operation

## Performance tuning

### Connection pooling

Configure HTTP transport settings for your workload:

```yaml
server:
  http:
    max_idle_conns: 512
    max_idle_conns_per_host: 128
    idle_conn_timeout_seconds: 90
```

More idle connections reduce connection establishment overhead but consume memory.

### Rate limiting scope

Rate limiting scope affects performance:

- `route` – fastest, single limiter per route
- `ip` – requires IP extraction, per-IP limiters
- `api_key` – requires auth context, per-key limiters

For high-throughput routes, consider `route` scope with appropriate limits.

### Caching

Enable caching for idempotent endpoints:

```yaml
routes:
  - id: products-api
    path: /api/products/*
    method: GET
    upstream_id: products
    cache:
      enable: true
      ttl_seconds: 300
```

Cache hits bypass the upstream entirely, providing significant latency reduction.

### Response headers

Minimize header manipulation for latency-sensitive routes:

```yaml
routes:
  - id: latency-critical
    path: /api/realtime/*
    method: GET
    upstream_id: realtime
    # No response_headers section = no processing overhead
```

## Comparison with Tyk

Veilgate is designed to be simpler and faster than Tyk by:

- **No embedded scripting** – No JSVM or Go plugins in the hot path
- **No external dependencies** – No Redis/Mongo required for core functionality
- **Minimal middleware** – Only the middleware you configure is executed
- **Efficient routing** – chi router with zero-allocation path matching
- **Shared transports** – Connection pooling shared across routes

Typical improvements over Tyk:
- 30-50% lower latency for pass-through proxy
- 60-80% lower memory footprint
- Faster cold starts (< 1s vs 5-10s)

## Load testing

For production capacity planning, use external load testing tools:

### Using hey

```bash
# Install hey
go install github.com/rakyll/hey@latest

# Basic load test
hey -n 10000 -c 100 http://localhost:8080/api/health

# With authentication
hey -n 10000 -c 100 -H "X-API-Key: your-key" http://localhost:8080/api/data
```

### Using wrk

```bash
# Basic test
wrk -t4 -c100 -d30s http://localhost:8080/api/health

# With script for POST requests
wrk -t4 -c100 -d30s -s post.lua http://localhost:8080/api/data
```

### Using vegeta

```bash
# Create target file
echo "GET http://localhost:8080/api/health" | vegeta attack -rate=1000 -duration=30s | vegeta report
```

## Profiling

### CPU profiling

```bash
# Enable pprof in admin server (if configured)
go tool pprof http://localhost:9090/debug/pprof/profile?seconds=30

# From benchmark
go test -bench=BenchmarkGatewayRoundTrip -cpuprofile=cpu.out ./internal/perf/...
go tool pprof cpu.out
```

### Memory profiling

```bash
go test -bench=BenchmarkGatewayRoundTrip -memprofile=mem.out ./internal/perf/...
go tool pprof mem.out
```

### Trace

```bash
go test -bench=BenchmarkGatewayRoundTrip -trace=trace.out ./internal/perf/...
go tool trace trace.out
```

## Production monitoring

Monitor these metrics in production:

- `veilgate_http_request_duration_seconds` – Request latency histogram
- `veilgate_http_requests_total` – Request count by route and status
- `veilgate_upstream_health` – Backend health status
- `veilgate_ratelimit_rejections_total` – Rate limit events
- Process metrics (CPU, memory, goroutines)

Set alerts for:
- p99 latency exceeding SLO
- High rate limit rejection rate
- Upstream health degradation
- Memory growth (potential leak)
