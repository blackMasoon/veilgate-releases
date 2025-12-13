---
title: Dashboard guide
description: Complete guide to using the Veilgate admin dashboard for monitoring and configuration.
---

# Dashboard guide

The Veilgate dashboard provides a web-based interface for monitoring gateway health,
managing routes, and configuring security policies.

## Accessing the dashboard

The dashboard is served by the admin server (default port 9090):

```
http://localhost:9090/dashboard
```

If admin authentication is configured, you'll be prompted to log in first.

## Overview page

The Overview page displays:

- **Gateway status**: Health and readiness indicators
- **Quick stats**: Total routes, APIs, and active API keys
- **Top routes**: Routes with the most traffic
- **Top API keys**: Most active API keys by request count

This page auto-refreshes to show current statistics.

## Routes view

The Routes view allows you to manage all gateway routes.

### Viewing routes

The routes table displays:

| Column | Description |
|--------|-------------|
| ID | Unique route identifier |
| Method | HTTP method(s) matched |
| Host | Optional host matcher |
| Path | URL path pattern |
| API | Target backend API |
| Auth | Authentication requirements |
| Rate Limit | Rate limiting configuration |
| Features | Active features (CORS, cache, rewrite, etc.) |

### Creating a route

1. Click **Add route**
2. Fill in the required fields:
   - **ID**: Unique identifier (e.g., `orders-api-v2`)
   - **Method**: HTTP method or `*` for all
   - **Path**: URL pattern (e.g., `/api/v1/*`)
   - **API ID**: Target backend API
3. Configure optional features:
   - **Rewrite**: Path prefix stripping/adding
   - **CORS**: Cross-origin settings
   - **Response Headers**: Add/remove headers
   - **Cache**: Response caching
   - **Proxy**: Host and TLS options
   - **IP Filter**: CIDR-based access control
4. Click **Create route**

### Editing a route

1. Click **Edit** next to the route
2. Modify the desired fields
3. Click **Save changes**

Note: Route ID cannot be changed after creation.

### Deleting a route

1. Click **Delete** next to the route
2. Confirm the deletion

Warning: Deletion is immediate and cannot be undone.

### Route features

The Features column shows pills for active features:

- **rewrite**: Path rewriting is configured
- **CORS**: Cross-origin requests enabled
- **headers**: Response header manipulation
- **cache**: Response caching enabled
- **proxy**: Custom proxy settings
- **IP filter**: IP-based access control

## APIs view

Manage backend API services in the APIs view.

### Viewing APIs

The table shows:

| Column | Description |
|--------|-------------|
| API ID | Unique API identifier |
| Target URLs | Backend server URLs |

### Creating an API

1. Click **Add API**
2. Enter the **API ID**
3. Add target URLs (one per line)
4. Click **Create API**

### Editing an API

1. Click **Edit** next to the API
2. Modify target URLs
3. Click **Save changes**

### Deleting an API

1. Click **Delete** next to the API
2. Confirm the deletion

Note: You cannot delete an API that is referenced by routes.

### Managing API Routes

From the API Detail View, you can manage routes specifically for that API:

1. Click on an API ID to open the detail view
2. Switch to the **Routes** tab
3. You can perform the following actions:
   - **Create Route**: Click **Create Route** to add a new route. The "API ID" will be automatically set to the current API.
   - **Edit Route**: Click **Edit** next to a route to modify its configuration.
   - **Delete Route**: Click **Delete** to remove a route.

## Policies view

The Policies view manages gateway-wide authentication defaults, the fallback rate limit, and the credentials/routes can reference (API keys and JWT issuers). Routes can override these defaults per route, but they inherit them when nothing is set locally.

### API Keys section

Displays configured API keys (both config-defined and generated in the UI) with:

- ID and label
- Active/inactive status
- Default header name (global setting used by API key auth)

### Defaults section

- Global API key header used when routes enable API-key auth
- Default rate limit applied when a route has no per-route limit configured

### JWT Issuers section

Shows configured JWT providers:

- Issuer ID and URL
- Algorithm (HS256, RS256, JWKS)
- Audience constraints
- JWKS URL and cache TTL (for JWKS)

### Managing API keys

API keys are credentials, not policies. Use them when a route requires API key auth; clients must send the key in the configured header.

#### Generating a new key

1. Optionally enter a **Label**
2. Click **Generate API key**
3. **Copy the key immediately** – it won't be shown again

#### Activating/Deactivating keys

1. Find the key in the table
2. Click **Activate** or **Deactivate**

#### Deleting keys

1. Click **Delete** next to the key
2. Confirm the deletion

## MCP Tools view

Manage Model Context Protocol (MCP) tool definitions.

### Current tools

Lists configured MCP tools with:

- Name and description
- Linked route ID
- HTTP method and path

### Design tools

Generate MCP tool configurations:

1. Select routes to expose as tools
2. Add descriptions
3. Export configuration snippet

### Export MCP server

Generate a complete MCP server project:

1. Select tools to include
2. Configure server metadata
3. Download as ZIP

## Security Metrics view

Monitor authentication and rate limiting events.

### Auth failures

Shows authentication failure counts by:

- Route ID
- Failure reason (API key, JWT, etc.)

### Rate limit rejections

Displays rate limiting events by:

- Route ID
- Scope (IP, API key, route)

## Dashboard authentication

When `admin_auth` is configured, the dashboard requires login.

### Logging in

1. Navigate to `/dashboard`
2. Enter username and password
3. Click **Login**

The session is stored in an HTTP-only cookie.

### Logging out

1. Click **Logout** in the navigation
2. Confirm logout

## Keyboard shortcuts

| Key | Action |
|-----|--------|
| `r` | Refresh current view |
| `h` | Go to Overview |
| `?` | Show help |

## Troubleshooting

### Dashboard not loading

1. Check that the admin server is running on the correct port
2. Verify no firewall rules blocking access
3. Check browser console for errors

### Login issues

1. Verify admin credentials are configured
2. Check that cookies are enabled
3. Try clearing browser cookies

### Data not updating

1. Click refresh or press `r`
2. Check gateway health status
3. Verify admin API is responding

### CORS errors in browser

The dashboard makes requests to the admin API on the same origin.
If you're running the dashboard from a different origin:

1. Configure CORS on the admin server
2. Or access via the same origin

## API access

All dashboard functionality is backed by the admin JSON API.
You can use these endpoints directly for automation:

```bash
# List routes
curl http://localhost:9090/admin/routes

# Create route
curl -X POST http://localhost:9090/admin/routes \
  -H "Content-Type: application/json" \
  -d '{"id":"new-route","path":"/new/*","method":"*","upstream_id":"backend"}'

# Get statistics
curl http://localhost:9090/admin/stats/routes
```

See the [Admin API reference](../reference/admin-api.md) for complete documentation.
