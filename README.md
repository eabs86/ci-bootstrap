# ⚡ ci-bootstrap

**One skill. Any stack. A production-grade pipeline: pre-commit → CI → CD.**

`ci-bootstrap` is an [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) for
[Claude Code](https://claude.com/claude-code) that turns *"we should really set up CI someday"*
into a pull request. It inspects **your** repository, detects **your** stack and conventions, and
builds the whole quality pipeline around what already exists — instead of pasting someone else's
template on top of it.

```
you:    /ci-bootstrap
claude: 🔍 Found: Python (backend/, ruff already configured), TypeScript (frontend/, eslint 9),
        Postgres in docker-compose, branches main + develop, no existing CI.
        Autopilot or Guided?
you:    autopilot
claude: 🚀 ...opens a PR with pre-commit hooks, ci.yml, ci-security.yml and a deploy skeleton.
```

---

## Why this exists

Every CI tutorial assumes a fresh repo with one language and no history. Real repos are messy:
a Python API here, a React app there, a `flake8` config nobody remembers writing, a `develop`
branch with unclear status, and a bash hook that breaks on your teammate's Windows machine.

Copy-pasted pipeline templates fail in the same five ways every time. This skill encodes the
fixes as **non-negotiable principles**:

| # | Principle | The failure it prevents |
|---|-----------|------------------------|
| 1 | **Version parity** — every tool pinned to the *same* version in pre-commit and CI | "Passes locally, fails in CI" (and vice versa) |
| 2 | **Cost gating** — PRs run fast checks only; the heavy suite runs on push to integration/production | 40-minute PR feedback loops that make people stop opening PRs |
| 3 | **Smart concurrency** — new pushes cancel stale runs, *except on production* | Cancelled production deploys, wasted runner minutes |
| 4 | **Path scoping** — each linter only sees its own package | The frontend linter screaming about the backend |
| 5 | **Cross-platform hooks** — Node wrapper scripts, never `bash -c` | Hooks that break for every Windows contributor |
| 6 | **Surgical changes** — extend existing CI, never rewrite it | The PR nobody dares to review because it touches everything |
| 7 | **No invented credentials** — unclear deploy target ⇒ disabled skeleton + TODOs | Secrets hallucinated into YAML |
| 8 | **Reuse before replace** — your tools stay unless *you* decide otherwise | Waking up to find flake8 silently replaced by something else |

## Two modes

| | 🚀 **Autopilot** | 🧭 **Guided** |
|---|---|---|
| **Flow** | All phases end to end | Pause + confirm after every phase |
| **Ambiguity** | Resolved with documented defaults, reported at the end | Turned into questions for you |
| **Stops only for** | Genuinely blocking decisions (e.g., can't tell which branch is production) | Every checkpoint |
| **Best for** | Solo projects, prototypes, "just make it good" | Team repos, existing CI, strong opinions |

Pick one when invoking (`/ci-bootstrap autopilot`), or the skill asks first.
**In both modes, discovery is strictly read-only** — nothing is written before the stack is understood.

## What it builds

```
Phase 0  🔍 Discovery      — detect languages, package roots, existing tools & versions,
                             branch model, test services, deploy targets. Read-only.
Phase 1  🪝 Pre-commit     — hygiene hooks, per-language format/lint/security scoped by path,
                             Conventional Commits, optional domain guard scripts.
Phase 2  ⚙️  CI             — ci.yml with per-layer quality jobs, native caching, service
                             containers (Postgres/Redis/...), Docker build checks, plus a
                             ci-security.yml with secret scanning and weekly dependency audits.
Phase 3  🚢 CD             — deploy workflows chained to green CI (workflow_run), production
                             deploys on release. Activated only with a confirmed target;
                             otherwise delivered as safe, disabled skeletons.
Phase 4  ✅ Validation      — pre-commit run --all-files, actionlint, version-parity audit,
                             PR against the integration branch with an honest description.
```

## Supported stacks

The skill detects — and adapts to — whatever it finds:

**Python** (ruff *or* your existing black/isort/flake8, bandit, pytest, pip-audit) ·
**JS/TS** (eslint, prettier, vitest/jest, npm audit) · **Go** (gofmt, go vet, golangci-lint,
govulncheck) · **Rust** (rustfmt, clippy, cargo audit) · **Java/Kotlin** · **.NET** · **Ruby** ·
**PHP** · **Dart/Flutter** · **Elixir** — including **polyglot monorepos** mixing several of the
above, each scoped to its own path.

Not listed? The skill falls back to the language's standard formatter/linter/test conventions.

## Install

**Per project** (share it with your team via git):

```bash
git clone https://github.com/eabs86/ci-bootstrap .claude/skills/ci-bootstrap
```

**Personal** (available in every project on your machine):

```bash
git clone https://github.com/eabs86/ci-bootstrap ~/.claude/skills/ci-bootstrap
```

No build step, no dependencies — a skill is just markdown.

## Use

```
/ci-bootstrap              # asks Autopilot vs Guided
/ci-bootstrap autopilot    # hands-off, end to end
/ci-bootstrap guided       # step-by-step with checkpoints
```

Or just say it in plain language — *"set up pre-commit and CI/CD for this repo"* — and Claude
Code will pick the skill up automatically.

## FAQ

**Will it overwrite my existing CI?**
No. Existing workflows are read, respected, and *extended*. Principle 6 is surgical changes;
anything questionable is pointed out, never deleted.

**Does a green PR mean everything passed?**
No — and the skill is honest about it. Heavy suites are gated to pushes on the integration and
production branches to keep PR feedback fast. This is stated in a YAML comment *and* in the PR
description it writes.

**Can it deploy my app?**
If discovery finds a clear target (Dockerfile + VPS secrets, `fly.toml`, `render.yaml`, ...), yes —
via workflows that only run after CI is green. If the target is unclear, you get a disabled
`workflow_dispatch` skeleton with documented TODOs. It will never invent credentials.

**Does it work on Windows?**
Yes — cross-platform hooks are one of the non-negotiable principles (Node wrappers instead of
bash one-liners).

**I use GitLab / another CI.**
Phase 2/3 are GitHub Actions today. Discovery and pre-commit (Phases 0–1) are CI-agnostic.
PRs welcome. 🙂

## Repository layout

```
ci-bootstrap/
├── SKILL.md                    # the skill: mode selection + principles + phase orchestration
├── references/
│   ├── discovery.md            # Phase 0 — stack & convention detection
│   ├── pre-commit.md           # Phase 1 — hooks per language, guards, Conventional Commits
│   ├── ci.md                   # Phase 2 — workflows, gating, services, security
│   ├── cd.md                   # Phase 3 — deploy patterns per target, secrets policy
│   └── validation.md           # Phase 4 — parity audit, actionlint, PR etiquette
├── README.md
└── LICENSE                     # MIT
```

## Contributing

Found a stack it handles poorly? A principle worth adding? Open an issue or PR — the whole skill
is plain markdown, so contributing is editing text.

## License

[MIT](LICENSE) — use it, fork it, ship it.
