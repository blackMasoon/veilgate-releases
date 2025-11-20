---
title: Changelog & Conventional Commits
description: How Veilgate generates its changelog and how to write commit messages that feed it.
---

# Changelog & Conventional Commits

Veilgate uses an **automated changelog** generated from git history using
the **Conventional Commits** convention. The CI release workflow reads
commits between versions and updates `CHANGELOG.md` and GitHub Releases
automatically.

## Conventional Commits in Veilgate

Commit messages SHOULD follow this shape:

```text
type(scope): short description
```

Where:

- `type` is one of:
  - `feat` – new user-facing feature.
  - `fix` – bug fix.
  - `perf` – performance improvements.
  - `refactor` – internal refactors that are not features or fixes.
  - `docs` – documentation-only changes.
  - `test` – tests only.
  - `build` – build tooling, dependencies.
  - `ci` – CI configuration, workflows.
  - `chore` – maintenance, small internal tweaks.
  - `revert` – reverting a previous commit.
- `scope` is optional and indicates the main area, e.g. `router`, `admin-api`,
  `dashboard`, `mcp`, `release`, `docs`.

Examples:

```text
feat(router): add host-based routing support
fix(ratelimit): avoid nil dereference when config is missing
docs: describe MCP tools in configuration reference
ci(release): make checkout fetch full history
```

### Breaking changes

Breaking changes SHOULD be marked using one of:

- A `!` after the type or scope:

  ```text
  feat!: remove deprecated gateway flags
  refactor(router)!: change default path matching semantics
  ```

- Or a `BREAKING CHANGE:` footer in the commit body:

  ```text
  feat(auth): switch default JWT algorithm to RS256

  BREAKING CHANGE: HS256 tokens are no longer accepted by default; config must be updated.
  ```

The changelog generator detects `!` and `BREAKING CHANGE:` and highlights
these entries accordingly.

### Release commits

Release/version bump commits use a dedicated form so they are easy to
exclude from the changelog:

```text
chore(release): vX.Y.Z
```

These commits are created automatically by the GitHub Actions release
workflow and SHOULD NOT be authored manually.

## How the changelog is generated

- The CI workflow `.github/workflows/release.yml`:
  - Bumps the `VERSION` file on pushes to `main`.
  - Runs `scripts/gen-changelog.sh` to:
    - Read commits between the previous tag (e.g. `v0.1.7`) and `HEAD`.
    - Group them into sections like **Added**, **Fixed**, **Changed**,
      **Performance**, **Docs**, **Tests**, **CI**, **Internal**.
    - Prepend a new `## [vX.Y.Z] - YYYY-MM-DD` section to `CHANGELOG.md`.
  - Extracts the section for the current version and uses it as the body
    of the GitHub Release.

- `CHANGELOG.md` is therefore **single source of truth for release notes**
  and is **auto-generated**. Manual edits will be overwritten.

## Local preview

You can preview what the next changelog section will look like without
modifying `CHANGELOG.md`:

```bash
scripts/gen-changelog.sh --from v0.1.7 --to HEAD --version 0.1.8 --preview
```

- `--from` should usually be the last released tag (`vX.Y.Z`).
- `--to` is typically `HEAD`.
- `--version` is the upcoming version number (without the leading `v`).

This command prints the generated section to stdout instead of writing it
into `CHANGELOG.md`.


