---
title: Functional tests (local and ZIP)
description: End-to-end scripts that validate Veilgate with a dev license and a release ZIP.
---

# Functional tests

Two end-to-end flows are available to validate the full stack, including issuing a license and running smoke tests.

## 1) Local functional test (images from repo)

Script: `scripts/func-test.sh`

- Builds Docker images for `veilgate`, `demo-upstream`, and `license-server` from the repo.
- Starts `license-server`, issues a development license, injects it into a config.
- Starts `demo-upstream` and `veilgate` and runs smoke tests:
  - GET `/` (JSON response)
  - CRUD on `/items`
- Tears down and cleans temporary artifacts.

Requirements: Docker, docker compose, `jq`, `curl`.

Run:

```bash
scripts/func-test.sh
```

## 2) Release ZIP functional test

Script: `scripts/func-test-zip.sh`

- Invokes `scripts/release.sh --bump none --no-push --compose-local` to create a release ZIP that references locally tagged images.
- Unzips the release, starts `license-server` separately, issues a license and injects it into the unzipped `config/veilgate.yaml`.
- Adds `VEILGATE_LICENSE_PUBKEY_B64` environment into the composed `veilgate` service and starts the stack from the ZIP.
- Runs the same smoke tests as the local functional test and tears down the stack.

Requirements: Docker, docker compose, `jq`, `curl`, `unzip`.

Run:

```bash
scripts/func-test-zip.sh
```

## CI integration

The GitHub Actions workflow (`.github/workflows/ci.yml`) runs both functional tests on pushes and PRs:

- `functional-test` – local images flow
- `functional-test-zip` – release ZIP flow (using `--no-push` and `--compose-local`)

These jobs depend on the `build-and-test` job to ensure unit tests pass before running end-to-end checks.

