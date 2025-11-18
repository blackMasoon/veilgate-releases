---
title: Contributing to documentation
description: How to update Veilgate docs, keep them in sync with code and validate builds.
---

# Contributing to documentation

This project treats the Markdown files under `docs/` as the single source of truth
for product and technical documentation.

## Where to put things

- `docs/overview/` – high-level product overview and positioning.
- `docs/concepts/` – core concepts such as routes, upstreams, policies, admin API, MCP.
- `docs/how-to/` – task-oriented guides (quickstart, securing routes, MCP, etc.).
- `docs/reference/` – detailed references (configuration, admin API, MCP config, CLI).

When adding or changing features:

1. Decide which section the change belongs to.
2. Add or update the relevant Markdown file(s), including frontmatter:

   ```yaml
   ---
   title: Title of the page
   description: One-line summary used by the docs site.
   ---
   ```

## Admin API and generated specs

The structured definition of the admin API is generated from code, not edited by hand.

- The generator lives in `internal/docsgen`.
- The CLI entrypoint is:

  ```bash
  go run ./cmd/veilgate docs export-admin-openapi \
    -out dist/docs-src/reference/admin-api.yaml \
    -version-file VERSION
  ```

When adding or changing admin endpoints:

1. Update the HTTP handlers in `internal/admin/admin.go`.
2. Update the endpoint list in `internal/docsgen` so OpenAPI stays in sync.
3. Adjust `docs/reference/admin-api.md` text if needed.

## Release and publishing flow

On release (via `.github/workflows/release.yml` + `scripts/release.sh`):

1. `scripts/release.sh` copies `docs/` into `dist/docs-src/` and runs the docs generator.
2. The release workflow syncs `dist/docs-src/` into the `blackMasoon/veilgate-releases` repo.
3. If `mkdocs.yml` exists in `veilgate-releases`, the workflow runs `mkdocs build` there.

The MkDocs configuration and navigation live in the `veilgate-releases` repository.
If you add a new page under `docs/`, remember to update the `mkdocs.yml` nav in that repo
so it appears in the public documentation site.

## Local validation (optional)

For local work you can:

- run `go vet ./...` and `go test ./...` as usual,
- run the docs generator command above to ensure OpenAPI still generates,
- run `mkdocs build` in a checkout of `veilgate-releases` to validate the full site.


