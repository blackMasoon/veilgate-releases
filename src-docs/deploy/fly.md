## Fly.io test deploy (gateway + portal + license admin)

This repository includes a CI workflow that deploys test instances to Fly.io on every merge to the default branch.

Components:
- `veilgate-test-gw` — gateway (ports 80/443) and admin/dashboard (port 9090)
- `veilgate-test-license` — license admin/portal (port 9400)

Both apps are configured as single-machine, scale-to-zero (auto-stop/auto-start) instances.

### Files
- `deploy/fly/veilgate.fly.toml` — Fly app config for gateway+portal (serves a minimal, license-free test config)
- `deploy/fly/license.fly.toml` — Fly app config for license admin
- `examples/fly/veilgate.yaml` — Minimal test config (<= 3 routes, 1 upstream)
- `.github/workflows/fly-deploy.yml` — GitHub Actions workflow

### Required repository secrets
Set these in your GitHub repository settings → Secrets and variables → Actions:

- `FLY_API_TOKEN` — Fly.io access token
- `FLY_ORG` — Fly.io organization slug (np. `personal`, albo własna nazwa org)

License server (admin + portal seed on first deploy):
- `LICENSE_ADMIN_TOKEN` — bearer token for JSON admin API
- `LICENSE_PORTAL_ADMIN_USER` — initial portal admin username
- `LICENSE_PORTAL_ADMIN_PASSWORD` — initial portal admin password
- `LICENSE_SESSION_SECRET` — HMAC secret for portal sessions

Veilgate admin (dashboard login seed on first deploy):
- `VEILGATE_DEV_ADMIN_USER` — initial admin username
- `VEILGATE_DEV_ADMIN_PASSWORD` — initial admin password
- `VEILGATE_ADMIN_SESSION_SECRET` — session secret for admin auth

Note: the gateway test config intentionally stays within demo limits (no license required).
If you want to validate enterprise limits, you can:
1. Issue a license via the license server (`POST /issue` with `LICENSE_ADMIN_TOKEN`)
2. Bake the token into a config file and point the Fly command to it (or add a new image variant that copies the licensed config).

### Access
- Domyślne nazwy aplikacji to `veilgate-test-gw` i `veilgate-test-license`. Możesz je nadpisać repozytoryjnymi Variables `FLY_GW_APP` i `FLY_LICENSE_APP`.
- Gateway: `https://<GW_APP>.fly.dev/`
- Admin/dashboard: `https://<GW_APP>.fly.dev:9090/dashboard` (login z VEILGATE_DEV_*)
- License portal: `https://<LICENSE_APP>.fly.dev:9400/portal`

### Scale-to-zero
Both apps in `fly.toml` have:
- `auto_stop_machines = "stop"`
- `auto_start_machines = true`
- `min_machines_running = 0`

This keeps costs minimal and wakes machines on incoming requests.
