# Phase 4 — Validation & delivery

## 1. Run the hooks for real

```
pre-commit run --all-files
```

Fix what the hooks flag. Auto-fixes (formatting, EOF, whitespace) are safe to keep; anything that
requires a judgment call (a lint rule fighting existing code style, a bandit finding in legacy
code) gets reported, not silently suppressed. If a rule is clearly wrong for this repo, scope it
down in config with a comment explaining why — don't blanket-disable.

## 2. Validate the workflows

- Run `actionlint` if available (`actionlint` binary, or
  `docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:latest -color` if Docker is present).
  Not available → do a careful manual review of the YAML and say that actionlint was skipped.
- Re-check the two invariants by reading the final YAML:
  - **Gating**: no heavy job runs on `pull_request`; the heavy `if:` matches exactly the
    integration/production push condition.
  - **Concurrency**: production ref never has `cancel-in-progress: true`.

## 3. Version parity audit

Produce the side-by-side table: tool → version in `.pre-commit-config.yaml` → version in CI.
Every row must match. Include the table in the final summary.

## 4. Branch & PR

- Create a feature branch (e.g., `ci/bootstrap-pipeline`) — never commit directly to integration
  or production.
- Open the PR against the **integration branch** (never production). Trunk-based repos: against
  the trunk, which is the only branch.
- The PR description must state explicitly: **"The full test suite does NOT run on this PR — it
  runs on merge to `<integration>`/`<production>`."**
- No AI signatures in commits or PR description.
- Only commit/push/open the PR if the user asked for it or confirmed it at a checkpoint; otherwise
  leave the working tree ready and provide the exact commands.

## 5. Final summary (both modes)

One consolidated report:

1. **Files created/modified** — grouped by phase.
2. **Decisions taken** — including every Autopilot default that resolved an ambiguity.
3. **Version parity table** (from step 3).
4. **Secrets to register** — name → where to get it → which workflow uses it (from Phase 3).
5. **What is intentionally NOT active** — disabled deploy skeletons, heavy suites gated off PRs,
   guards not created because no invariant existed.
6. **Next steps for the user** — register secrets, enable branch protection with the new checks,
   run `hooks:install`, merge order.
