# Phase 0 — Discovery (read-only; change nothing yet)

Investigate the repository and produce a short summary of what you found. **Do not write or modify
any file in this phase**, in any mode.

## What to detect

### 1. Languages and package managers, per directory

Look for manifests and lockfiles. Map each hit to its ecosystem and toolchain:

| Evidence | Ecosystem | Typical toolchain |
|----------|-----------|-------------------|
| `pyproject.toml`, `requirements*.txt`, `poetry.lock`, `Pipfile`, `uv.lock`, `setup.py` | Python | ruff / black+isort+flake8, mypy, pytest, bandit, pip-audit |
| `package.json`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `bun.lockb` | JS/TS | eslint, prettier, tsc, vitest/jest, npm audit |
| `go.mod`, `go.sum` | Go | gofmt, go vet, golangci-lint, go test, govulncheck |
| `Cargo.toml`, `Cargo.lock` | Rust | rustfmt, clippy, cargo test, cargo audit |
| `pom.xml`, `build.gradle`, `build.gradle.kts` | Java/Kotlin | spotless/checkstyle, ktlint/detekt, JUnit |
| `*.csproj`, `*.sln`, `Directory.Build.props` | .NET | dotnet format, analyzers, dotnet test |
| `Gemfile`, `Gemfile.lock` | Ruby | rubocop, rspec, bundler-audit |
| `pubspec.yaml` | Dart/Flutter | dart format, dart analyze, flutter test |
| `composer.json` | PHP | php-cs-fixer, phpstan, phpunit |
| `mix.exs` | Elixir | mix format, credo, mix test |

Note the **version pins** already declared (`.python-version`, `.nvmrc`, `engines`,
`requires-python`, `.tool-versions`, `rust-toolchain.toml`) — CI must use the same runtimes.

### 2. Layout: single package or monorepo?

List the root of every service/package. Linter scope must match these paths exactly. Common
signals: `packages/`, `apps/`, `services/`, `backend/` + `frontend/`, pnpm/yarn/npm workspaces,
`turbo.json`, `nx.json`, `lerna.json`, multiple `pyproject.toml`.

### 3. Quality tools already present — and their versions

`.flake8`, `ruff.toml` / `[tool.ruff]`, `.eslintrc*` / `eslint.config.*`, `.prettierrc*`,
`pytest.ini` / `[tool.pytest]`, `vitest.config.*` / `jest.config.*`, `mypy.ini`, `tsconfig.json`,
`.editorconfig`, `.pre-commit-config.yaml`, `Makefile` / `justfile` / `Taskfile.yml` targets.

**Reuse what exists — do not duplicate, and do not swap tools without asking.** Record each tool's
pinned version: parity with CI depends on it.

### 4. Existing CI

Is there `.github/workflows/`? Another system (GitLab CI, CircleCI, Jenkins, Azure Pipelines)?
Read what exists. Do not delete anything out of scope; integrate with it. If a workflow already
covers a job you would create, extend it instead of shadowing it.

### 5. Branch model

Which branch is **integration** (`dev` / `develop` / `staging`) and which is **production**
(`main` / `master`)? Check remote branches (`git branch -r`), branch protection hints, existing
workflow triggers, and recent merge history. If there is only one branch, the model is
"trunk-based": PRs target production and there is no separate integration deploy.

- **Autopilot default**: if only `main` exists → trunk-based. If `main` + `develop`-like branch
  exist → the develop-like branch is integration. Anything else genuinely ambiguous → this is a
  *blocking question*, ask.
- **Guided**: always confirm the branch model.

### 6. Test services

Postgres / MySQL / Redis / message queues: detect via `docker-compose*.yml`, test settings, env
examples (`.env.example`), and dependency lists (psycopg, redis, amqplib, kafkajs, ...).
These become `services:` blocks in CI.

### 7. Deploy environment (if any)

Dockerfiles? SSH scripts to a VPS? Platform config (`fly.toml`, `render.yaml`, `vercel.json`,
`netlify.toml`, Helm charts, `app.yaml`)? Existing GitHub secrets referenced in old workflows?
If nothing is clear, Phase 3 delivers a **skeleton** with marked TODOs — never invented
credentials.

## Output of this phase

Present a compact summary:

1. Detected stacks and their roots (table).
2. Existing quality tools + versions to reuse.
3. Existing CI and how you will integrate with it.
4. Proposed branch model (integration / production).
5. Test services needed in CI.
6. Deploy target status (confirmed / skeleton-only).
7. **Your assumptions**, explicitly listed.
8. **Decisions that belong to the user**:
   - integration branch (if ambiguous),
   - tool ties (e.g., flake8 vs ruff when both are plausible),
   - whether to activate deploy (Phase 3) or keep it as a disabled skeleton.

**Guided mode**: stop here and wait for confirmation before Phase 1.
**Autopilot mode**: state the defaults you chose for each open point and continue — unless a
*blocking question* (ambiguous production branch, destructive conflict with existing CI) remains,
in which case ask only that question.
