---
title: MCP configuration
description: Reference for configuring MCP tools in Veilgate.
---

# MCP configuration

MCP (Model Context Protocol) support in Veilgate is driven by configuration.

At a high level, MCP configuration describes:

- tools exposed to LLM agents,
- mappings from tools to gateway routes / HTTP endpoints,
- JSON schemas for tool inputs and outputs.

## MCP section in `GatewayConfig`

MCP tools are configured under the top-level `mcp` section of `GatewayConfig`.

Each tool is represented by a flat structure that intentionally mirrors
Veilgate routes instead of introducing a separate policy system.

### Schema

In Go types (see `internal/config/types.go`):

```go
type MCPConfig struct {
    Tools []MCPToolSpec `yaml:"tools,omitempty" json:"tools,omitempty"`
}

type MCPToolSpec struct {
    Name        string `yaml:"name" json:"name"`
    Description string `yaml:"description,omitempty" json:"description,omitempty"`
    RouteID     string `yaml:"route_id,omitempty" json:"route_id,omitempty"`
    Method      string `yaml:"method,omitempty" json:"method,omitempty"`
    Path        string `yaml:"path,omitempty" json:"path,omitempty"`
}
```

- `name` – **required**, MCP tool identifier (must be unique across tools),
- `description` – optional, human-readable description,
- `route_id` – optional link to an existing `routes[].id` in `GatewayConfig`,
- `method` – optional HTTP method (e.g. `GET`, `POST`),
- `path` – optional HTTP path (e.g. `/users/{id}`).

Validation rules (see `internal/config/validate.go`):

- every tool must have `name`,
- tool names must be unique,
- if `route_id` is provided, it must point at a route defined in `routes`.

### Minimal example

```yaml
server:
  listen_address: "0.0.0.0:8080"
admin:
  listen_address: "0.0.0.0:9090"
logging:
  level: "info"
metrics:
  enabled: true

upstreams:
  - id: "example-api"
    endpoints:
      - url: "http://example.com"

routes:
  - id: "example-route"
    path: "/example"
    method: "GET"
    upstream_id: "example-api"

mcp:
  tools:
    - name: "example_route_tool"
      description: "Call the /example route via Veilgate."
      route_id: "example-route"
      method: "GET"
      path: "/example"
```

This configuration defines a single MCP tool `example_route_tool` which is
logically coupled to the `example-route` route.

## Admin API: `/admin/mcp/tools`

The admin API exposes the current MCP tools and supports basic CRUD via:

```text
GET /admin/mcp/tools
POST /admin/mcp/tools
PUT  /admin/mcp/tools/{name}
DELETE /admin/mcp/tools/{name}
```

`GET /admin/mcp/tools` returns a JSON array of objects matching `MCPToolSpec`:

```json
[
  {
    "name": "example_route_tool",
    "description": "Call the /example route via Veilgate.",
    "route_id": "example-route",
    "method": "GET",
    "path": "/example"
  }
]
```

This endpoint:

- is protected by admin auth and rate limiting, like the other `/admin/*` APIs,
- is consumed by the dashboard MCP view,
- should not expose secrets (it only reflects tool metadata).

## Dashboard integration

The built-in dashboard includes:

- an **MCP tools view** listing tools from `/admin/mcp/tools` and showing:
  - `name` and `description`,
  - `route_id`,
  - `method` and `path`;
- an **MCP config helper** that generates a YAML fragment with `mcp.tools`
  based on the current routes.

The helper uses `/admin/routes` to list routes and produces a minimal config
snippet such as:

```yaml
mcp:
  tools:
    - name: "route-one"
      description: ""
      route_id: "route-one"
      method: "GET"
      path: "/one"
```

Operators can copy this fragment into the main Veilgate config or into a
separate `mcp-tools.yaml` file and adjust names and descriptions as needed.

The dashboard **does not** write or reload config files directly; it only
assists with generating YAML, keeping the operational model simple and safe.

## MCP scaffolding via CLI

The `veilgate mcp init` command scaffolds a minimal MCP server project that can
call APIs behind Veilgate:

```bash
veilgate mcp init -dir ./my-mcp-server -name my-mcp -desc "My MCP server" \
  -base-url http://localhost:8080 -route-id example-route
```

Scaffolding produces:

- `main.go` – example MCP server using `internal/mcp`,
- `README.md` – run instructions and an example MCP config snippet,
- `mcp-tools.yaml` – a sample MCP configuration for Veilgate:

```yaml
mcp:
  tools:
    - name: "my-mcp_example"
      description: "My MCP server"
      route_id: "example-route"
      method: "GET"
      path: "/"
```

You can merge `mcp-tools.yaml` into your main Veilgate config (or load it as an
additional file, depending on your configuration tooling) and then adjust
`route_id`, `method` and `path` to point at real routes.

## Exporting MCP servers from the dashboard

In addition to manual scaffolding via CLI, you can design MCP tools in the
dashboard and export a ready-to-run MCP server as a ZIP archive.

### Workflow

1. Open the **MCP Tools** tab in the dashboard.
2. Use **Design tools** to:
   - create, edit and delete MCP tools,
   - compose an \"API surface\" by selecting multiple routes and generating
     tools for them.
3. Switch to **Export MCP server**:
   - choose which tools to include (by default all),
   - provide MCP server name, directory inside the ZIP and base URL,
   - click **Download MCP server ZIP**.

The ZIP archive contains:

- `main.go` – minimal MCP server using Veilgate's internal MCP library,
- `README.md` – build and run instructions, plus wiring notes,
- `mcp-tools.yaml` – a config snippet with the selected tools.

You can:

- build and run the MCP server binary alongside Veilgate,
- merge `mcp-tools.yaml` into your gateway configuration,
- configure your MCP-compatible client to talk to the new MCP server.

## Summary

- `GatewayConfig.mcp.tools` defines a flat list of MCP tools,
- `/admin/mcp/tools` exposes the effective tools to the dashboard and other
  automation and supports CRUD,
- `veilgate mcp init` and the dashboard's MCP server ZIP export make it easy to
  bootstrap and maintain MCP integration without adding runtime complexity to
  the gateway.


