---
name: ci-bootstrap
description: >-
  Bootstrap a production-grade quality pipeline — pre-commit hooks, CI, and CD
  (GitHub Actions) — for any repository. Detects the real stack (Python, JS/TS,
  Go, Rust, Java, .NET, Ruby, Dart, ...), reuses existing tools and conventions
  instead of replacing them, and enforces version parity between local hooks and
  CI. Offers two modes: Autopilot (fully automated, sensible defaults) and
  Guided (step-by-step with confirmation checkpoints). Use when the user asks to
  set up pre-commit, CI/CD, GitHub Actions, linting pipelines, quality gates,
  deploy workflows, or to "professionalize" a repository.
---

# CI Bootstrap

You are going to build a quality and delivery pipeline for **THIS repository**, inspired by a
mature polyglot-monorepo reference, but **adapted to the real stack and conventions found here**.
Do NOT copy technologies that don't exist in this project. **Detect, confirm, then implement.**

## Step 1 — Choose the operating mode

If the user passed a mode in the arguments (`autopilot`, `auto`, `guided`, `interactive`,
`step-by-step`), use it. Otherwise, ask before doing anything else:

> **How do you want to run this?**
>
> 1. **Autopilot** — I run all phases end to end with sensible defaults. I only stop if I hit a
>    decision that is genuinely ambiguous and risky (e.g., can't tell which branch is production).
> 2. **Guided** — I pause after each phase, show you what I found / what I'm about to write, and
>    wait for your confirmation before touching anything.

Mode rules:

- **Autopilot**: run Phases 0→4 without pausing. Resolve ambiguity with the defaults documented in
  `references/` (and say which default you picked in the final summary). Only stop for the
  *blocking questions* listed at the end of Phase 0 — and even then, first try to resolve them from
  evidence (remote branches, existing configs, commit history).
- **Guided**: at the end of each phase, present results and the plan for the next phase as a short
  summary plus concrete questions (use structured questions when available). Do not write any file
  before Phase 0 is confirmed.

In **both** modes, Phase 0 is strictly read-only.

## Step 2 — Execute the phases

Each phase has a detailed reference file. Read it when you reach that phase — not before.

| Phase | What | Reference |
|-------|------|-----------|
| 0 | Discovery (read-only stack & convention detection) | [references/discovery.md](references/discovery.md) |
| 1 | Pre-commit hooks (`.pre-commit-config.yaml`) | [references/pre-commit.md](references/pre-commit.md) |
| 2 | CI (`.github/workflows/ci.yml` + security workflow) | [references/ci.md](references/ci.md) |
| 3 | CD (deploy skeletons, only activated with confirmed targets) | [references/cd.md](references/cd.md) |
| 4 | Validation & delivery | [references/validation.md](references/validation.md) |

## Non-negotiable principles (the "why" behind the reference)

These apply in every mode, every phase. They are not stylistic preferences — each one prevents a
concrete failure mode.

1. **Version parity pre-commit ↔ CI.** Pin the SAME version of every tool in both places. A hook
   that passes locally and fails in CI (or vice versa) destroys trust in the pipeline.
2. **Gating by cost.**
   - On **PRs**: only fast checks — format check, lint, security scan, light unit tests.
   - The **heavy suite** runs **only on `push` to the integration and production branches**:
     ```yaml
     if: github.event_name == 'push' && (github.ref == 'refs/heads/<integration>' || github.ref == 'refs/heads/<production>')
     ```
     Make this EXPLICIT in a YAML comment and tell the user that **"a green PR does not guarantee
     the full suite passed"**.
3. **Concurrency.** `cancel-in-progress: ${{ github.ref != 'refs/heads/<production>' }}` —
   production runs are never cancelled.
4. **Path scoping.** Each linter only runs on the files of its own package/service
   (`files:` regex in pre-commit; `paths:` filters and `working-directory` in CI).
5. **Cross-platform hooks.** For local hooks that call binaries from `node_modules`, use a Node
   wrapper script (`node scripts/precommit-runner.mjs <tool>`), never `bash -c 'cd ... && npx ...'`
   — bash may be missing from PATH on Windows. Only applies if there is a JS frontend.
6. **Surgical changes.** Do not rewrite CI that works; add the minimum. Do not delete files outside
   the task's scope — point at problems, don't erase them.
7. **No invented credentials.** All secrets via `secrets.*`. If the deploy target is unclear,
   deliver disabled skeletons with documented TODOs instead of guessing.
8. **Reuse before replace.** If the repo already uses flake8, don't silently switch it to Ruff.
   Tool swaps are a user decision, never a default.

## Step 3 — Deliver

Finish with a single final summary (see `references/validation.md` for the checklist):
what was created, decisions and defaults taken, secrets the user must register, and next steps.
In Guided mode this is the last checkpoint; in Autopilot it is the only long report the user sees —
make it count.
