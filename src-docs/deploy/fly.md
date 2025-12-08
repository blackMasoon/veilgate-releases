## Fly.io test deploy (gateway + portal + license admin)

This repository includes a CI workflow that deploys test instances to Fly.io on every merge to the default branch.

Components:
- `veilgate-test-gw` — gateway (ports 80/443) and admin/dashboard (port 9090)
- `veilgate-test-license` — license admin/portal (port 9400)

Both apps are configured as single-machine, scale-to-zero (auto-stop/auto-start) instances.

### Files
- `fly.gw.toml` — Fly app config for gateway+portal (root-level, poprawny kontekst builda)
- `fly.license.toml` — Fly app config for license admin (root-level)
- `examples/fly/veilgate.yaml` — Minimal test config (<= 3 routes, 1 upstream)
- `.github/workflows/fly-deploy.yml` — GitHub Actions workflow

### Required repository secrets
Set these in your GitHub repository settings → Secrets and variables → Actions:

- `FLY_API_TOKEN` — Fly.io access token
- `FLY_ORG` — (opcjonalny) slug organizacji Fly; wymagany tylko, jeśli masz wiele orgów i chcesz wymusić konkretną

License server (admin + portal seed on first deploy):
- `LICENSE_ADMIN_TOKEN` — bearer token for JSON admin API
- `LICENSE_PORTAL_ADMIN_USER` — initial portal admin username
- `LICENSE_PORTAL_ADMIN_PASSWORD` — initial portal admin password
- `LICENSE_SESSION_SECRET` — HMAC secret for portal sessions

Veilgate admin (dashboard login seed on first deploy):
- `VEILGATE_DEV_ADMIN_USER` — initial admin username
- `VEILGATE_DEV_ADMIN_PASSWORD` — initial admin password
- `VEILGATE_ADMIN_SESSION_SECRET` — session secret for admin auth

The deployment workflow automatically issues a test license for the Veilgate instance after deploying the license server:
1. Waits for the license server to become ready
2. Fetches the public key from the license server
3. Issues a license with enterprise limits (100 upstreams, 1000 routes, 365 days validity)
4. Configures the Veilgate instance with the license via secrets

The license is bound to the instance ID matching the app name (default: `veilgate-test-gw`) and includes all enterprise features (jwt, ratelimit, oauth2, mtls).

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
