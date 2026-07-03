# Phase 1 — Pre-commit (`.pre-commit-config.yaml` at the repo root)

Build the local quality gate. Everything here must later be mirrored in CI **at the same pinned
versions** (Phase 2).

If a `.pre-commit-config.yaml` already exists, extend it surgically — keep the user's existing
hooks and revs unless they are broken.

## 1. Hygiene hooks (always)

From `pre-commit/pre-commit-hooks`, pinned to a recent stable rev:

- `trailing-whitespace`
- `end-of-file-fixer`
- `check-yaml` (add `--unsafe` if the repo uses custom YAML tags, e.g. CloudFormation, Gitlab CI)
- `check-json`
- `check-toml`
- `check-added-large-files` with `--maxkb=1000`
- `check-merge-conflict`
- `check-case-conflict`
- `mixed-line-ending` with `--fix=lf`
- `detect-private-key`

## 2. Per detected language (only the ones that actually exist)

### Python

Formatter + import sort + lint + `bandit -ll`. **Use what the repo already adopts.** With no
existing preference, prefer **Ruff** (format + lint + import sorting in one tool); if
black/isort/flake8 are already configured, keep them.

- Scope with `files: ^<dir>/`.
- Multiple services with different configs → one hook entry per service, each scoped to its path.
- Type checking (mypy/pyright) only if the repo already runs it — it's too slow to introduce
  silently as a commit hook.

### JS / TS / Vue / React / Svelte

`eslint --fix` and `prettier --write` as **local hooks** using the project's own `node_modules`
(exact parity with CI — the mirrored-hook versions on pre-commit.ci mirrors drift).

Cross-platform rule: hooks must go through a Node wrapper, not bash. Create
`scripts/precommit-runner.mjs`:

```js
#!/usr/bin/env node
// Cross-platform pre-commit runner: resolves binaries from the nearest node_modules.
// Usage: node scripts/precommit-runner.mjs <package-dir> <tool> [...args]
import { spawnSync } from 'node:child_process';
import { resolve } from 'node:path';

const [dir, tool, ...args] = process.argv.slice(2);
const bin = resolve(dir, 'node_modules', '.bin', process.platform === 'win32' ? `${tool}.cmd` : tool);
const result = spawnSync(bin, args, { stdio: 'inherit', cwd: resolve(dir), shell: process.platform === 'win32' });
process.exit(result.status ?? 1);
```

Hook shape:

```yaml
- id: eslint-frontend
  name: eslint (frontend)
  language: system
  entry: node scripts/precommit-runner.mjs frontend eslint --fix
  files: ^frontend/.*\.(js|jsx|ts|tsx|vue)$
  pass_filenames: true
```

### Go

`gofmt` / `goimports`, `go vet`; `golangci-lint` if configured.

### Rust

`cargo fmt --check` as check hook; `cargo clippy -- -D warnings` (may be CI-only if slow).

### Other ecosystems

Use the language's standard formatter + linter (see the table in `discovery.md`). Keep commit-time
hooks fast (< ~5s per hook on a typical change); push anything slower to CI.

## 3. Conventional Commits

`commitizen` (Python-centric repos) or `commitlint` (JS-centric repos) on the `commit-msg` stage,
plus its config file (`[tool.commitizen]` in `pyproject.toml`, or `.commitlintrc.json`) with:

- the project's actual types/scopes (inspect `git log` for the dominant convention first — if the
  history already follows a pattern, encode that pattern, don't fight it),
- `header-max-length` (default 100).

## 4. Global excludes

One global `exclude:` regex for generated artifacts: `dist/`, `build/`, `.venv/`, `node_modules/`,
migration directories, `*.min.*`, vendored code, lockfiles (hygiene hooks like end-of-file-fixer
would churn them).

## 5. Top-level settings

```yaml
fail_fast: false
minimum_pre_commit_version: "3.5.0"
default_language_version:
  python: python3.12   # match the repo's pinned runtime
```

## 6. Domain guard gates (optional — only if a real invariant exists)

If the repo has cross-file invariants (constants that must stay in sync across services, secrets
that must never appear in logs, API schemas mirrored in two places), create a deterministic script
(AST parse / regex, **no runtime imports**) as a local hook:

```yaml
- id: guard-sync-constants
  name: guard: shared constants in sync
  language: system
  entry: python scripts/guards/check_constants_sync.py
  pass_filenames: false
```

MIRROR every guard as a CI job (`repo-guards`, Phase 2). Do not create guards speculatively —
only when Phase 0 found an actual invariant.

## 7. Convenience scripts

In the root `package.json` (JS repos) or `Makefile`/`justfile` (otherwise):

- `hooks:install` → `pre-commit install --install-hooks -t pre-commit -t commit-msg`
- `hooks:run` → `pre-commit run --all-files`
- `hooks:update` → `pre-commit autoupdate`

## Checkpoint

**Guided mode**: show the full proposed `.pre-commit-config.yaml` (and wrapper/guard scripts)
before writing, then write after confirmation and run `pre-commit run --all-files`.
**Autopilot mode**: write, run `pre-commit run --all-files`, commit auto-fixes if the user asked
for commits, and report anything the hooks flagged that needs a human decision.
