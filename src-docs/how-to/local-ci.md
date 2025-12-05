---
title: Local CI (pre-push checks)
description: Run unit + functional tests locally with one command, and install a pre-push hook.
---

# Local CI

This repo provides scripts and Make targets to run the same checks locally that CI runs.

## Quick start

Requirements:

- Go (>= 1.23)
- Docker + docker compose
- `curl`, `jq`, `unzip` (for functional tests)

Run everything:

```bash
make local-ci
```

This runs:

- `go mod tidy`
- `go test ./...`
- Build Docker images (alpine runtime)
- `scripts/func-test.sh` (local images functional E2E)
- `scripts/func-test-zip.sh` (release ZIP functional E2E)

## Pre-push hook (optional)

Install a Git pre-push hook to automatically run unit + functional tests before pushing:

```bash
scripts/install-git-hooks.sh
```

On every `git push`, the hook runs:

- `go test ./...`
- `scripts/func-test.sh`

If Docker is not available, it skips the functional test.

## Run the exact CI workflow locally (optional)

You can use `act` (https://github.com/nektos/act) to run GitHub Actions locally:

```bash
act -j functional-test
act -j functional-test-zip
```

Note: you may need to set up a compatible Docker runner image and install `jq`, `curl`, and `unzip` inside it.

