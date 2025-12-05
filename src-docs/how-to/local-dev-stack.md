---
title: Local dev stack (with license server)
description: Run Veilgate + license server + demo upstream locally using docker-compose and a helper script.
---

# Local dev stack

This repository includes a helper script to run a complete local stack:

- License server (Ed25519 keys, issue/verify/revoke)
- Veilgate gateway (with license verification enabled)
- Demo upstream service

## Prerequisites

- Docker and docker compose
- `curl` and `jq` on the host

## Start the stack

From the repo root:

```bash
scripts/run-local-stack.sh
```

What it does:

- Builds images for `license-server`, `demo-upstream` and `veilgate`
- Starts the license server on `localhost:9400`
- Fetches the public key and issues a development license
- Writes `.env.license` with `VEILGATE_LICENSE_PUBKEY_B64`
- Generates `dist/local/config.yaml` by injecting the license token
- Starts `demo-upstream` and `veilgate` with the generated config

Test calls:

```bash
curl -H 'X-API-Key: demo-secret-key' http://localhost:8080/
```

License server endpoints:

- `http://localhost:9400/pubkey`
- `POST http://localhost:9400/issue` (requires `LICENSE_ADMIN_TOKEN`, default `dev-admin-token`)
- `POST http://localhost:9400/verify`
- `POST http://localhost:9400/revoke` (admin)
- `http://localhost:9400/crl`

Admin portal:

- URL: `http://localhost:9400/portal`
- Default user (dev): `admin` / `dev-portal-pass`

## Customizing

Environment variables recognized by the script:

- `ADMIN_TOKEN` – admin bearer for issuing licenses (default `dev-admin-token`)
- `LICENSE_SUBJECT` – subject shown in issued licenses (default `Local Dev`)
- `MAX_UPSTREAMS` / `MAX_ROUTES` – entitlements
- `EXPIRES_DAYS` – validity period
- `INSTANCE_ID` – binds the license to this gateway instance id
