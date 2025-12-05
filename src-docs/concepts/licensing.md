---
title: Enterprise licensing
description: Offline-signed licenses with optional online validation and strict config enforcement.
---

# Enterprise licensing

Veilgate Enterprise uses Ed25519-signed tokens to convey entitlements and bind
them to a deployment identity. The gateway verifies licenses offline using a
built-in public key (overridable via `VEILGATE_LICENSE_PUBKEY_B64`) and enforces
limits early during startup and on admin-driven config mutations.

- Without a valid license: max 1 upstream, max 3 routes (demo mode).
- With a valid license: the license payload specifies `max_upstreams` and `max_routes`.
- Optional bindings: `instance_id` and `domains[]` allow scoping a license to a
  specific gateway instance or hostnames.
- Optional expiry: `exp` enforced strictly; there is no silent grace period.

Token format: `VEIL-1.<base64url(payload JSON)>.<base64url(signature)>`

- signature = Ed25519(private_key, base64url(payload))
- verification = Ed25519(public_key, base64url(payload))

The repository includes a minimal license server (`license-server/`) to generate
and manage licenses. In production, the public key must be provisioned into the
Veilgate runtime via `VEILGATE_LICENSE_PUBKEY_B64`, and the private key must be
protected by your issuance infrastructure.

Security hardening notes:

- Verification happens before router/materialization; invalid or insufficient
  licenses fail the process fast. The same checks apply on admin config mutate.
- Tokens are small, offline-verifiable, and auditable (JSON payload).
- A CRL endpoint (`/crl`) is provided by the sample server; integrating periodic
  revocation checks is possible without blocking startup.
- No secrets ship in the gateway binary; only a public key is needed.

