# Phase 2 — CI (`.github/workflows/ci.yml` + `ci-security.yml`)

Mirror Phase 1 in GitHub Actions, add tests and builds, and enforce the cost-gating and
concurrency principles. If CI already exists, extend it — do not shadow or replace working jobs.

`<integration>` and `<production>` below are the branches confirmed in Phase 0. In a trunk-based
repo there is only `<production>`; drop the integration entries.

## Skeleton

```yaml
name: CI

on:
  pull_request:
    branches: [<production>, <integration>]
  push:
    branches: [<integration>, <production>]

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  # Production pushes are never cancelled; everything else is superseded by newer runs.
  cancel-in-progress: ${{ github.ref != 'refs/heads/<production>' }}

env:
  # Pin runtimes to the versions detected in Phase 0.
  PYTHON_VERSION: "3.12"
  NODE_VERSION: "20"
```

## Jobs — one per layer; `matrix` when N services share a shape

### `<lang>-quality` (one per detected language/service)

checkout → setup with **native cache** (`actions/setup-python` `cache: pip`,
`actions/setup-node` `cache: npm`, `actions/setup-go`, `Swatinem/rust-cache`, ...) → install tools
**at versions identical to pre-commit** → then:

1. format `--check` (never `--fix` in CI),
2. lint,
3. security (`bandit -ll`, `eslint-plugin-security`, `gosec`, ... whatever matches the stack),
4. **tests** — light unit tests may run on PRs; the heavy suite goes behind the gate:

```yaml
      # ⚠️ COST GATE: the full/heavy suite runs ONLY on push to <integration> or <production>.
      # A green PR does NOT mean the full suite passed.
      - name: Full test suite
        if: github.event_name == 'push' && (github.ref == 'refs/heads/<integration>' || github.ref == 'refs/heads/<production>')
        run: <heavy test command>
```

Scope each job with `paths:` filters or a `working-directory` default so a docs-only change
doesn't run every suite. When using `paths:` on required checks, remember GitHub treats skipped
required checks as pending — prefer job-level `dorny/paths-filter` or keep required checks
unfiltered and fast.

### Tests that need a database/cache

Use `services:` with healthchecks, and export URLs:

```yaml
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U test"
          --health-interval 5s --health-timeout 5s --health-retries 10
      redis:
        image: redis:7
        ports: ["6379:6379"]
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s --health-timeout 5s --health-retries 10
    env:
      DATABASE_URL: postgresql://test:test@localhost:5432/test
      REDIS_URL: redis://localhost:6379/0
```

### `repo-guards` (only if Phase 1 created guard gates)

Runs the exact same guard scripts from Phase 1. Same script file, same interpreter — never a
reimplementation.

### `docker-build` (only if Dockerfiles exist)

Matrix over the Dockerfiles; `docker build` **without push** (push belongs to CD). Use
`docker/setup-buildx-action` + `docker/build-push-action` with `push: false` and GHA cache
(`cache-from: type=gha`, `cache-to: type=gha,mode=max`).

## Security workflow (`ci-security.yml`)

Separate workflow with three triggers:

```yaml
on:
  pull_request:
  schedule:
    - cron: "0 6 * * 1"   # weekly, Monday 06:00 UTC
  workflow_dispatch:
```

Jobs:

1. **Secret/PII gate on diffs** (PRs, if applicable): `gitleaks` or equivalent on the changed
   range.
2. **Static security baseline**: `bandit` at HIGH severity (Python), or the stack's equivalent —
   fails the PR only on HIGH; the full report goes to the scheduled run.
3. **Dependency scan** (scheduled + dispatch): `pip-audit` / `npm audit --audit-level=high` /
   `cargo audit` / `govulncheck` / `trivy fs .` — pick per detected stack.

## Version parity table

Before finishing this phase, build a small table (tool → pre-commit rev → CI version) and verify
every row matches. This table goes into the final summary.

## Checkpoint

**Guided mode**: show the proposed workflows and the parity table before writing.
**Autopilot mode**: write, then validate with `actionlint` if available (see Phase 4).
