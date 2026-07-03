# Phase 3 — CD (skeleton first; activate only with confirmed target + secrets)

Never invent credentials or deploy targets. Three possible states, decided in Phase 0:

- **Confirmed target + secrets named by the user** → deliver working workflows.
- **Target known, secrets not yet registered** → deliver working workflows and a checklist of the
  exact secret names the user must create.
- **No clear target** → deliver the workflows as `workflow_dispatch`-only (effectively disabled)
  with documented `# TODO:` markers at every point that needs a real value.

## Integration deploy — `deploy-<integration>.yml`

Triggered by the CI workflow completing successfully on the integration branch, never by push
directly (so a deploy can never outrun a red CI):

```yaml
name: Deploy (integration)

on:
  workflow_run:
    workflows: [CI]
    types: [completed]

concurrency:
  group: deploy-<integration>
  cancel-in-progress: true   # a newer integration deploy supersedes an in-flight one

jobs:
  deploy:
    if: >-
      github.event.workflow_run.conclusion == 'success' &&
      github.event.workflow_run.event == 'push' &&
      github.event.workflow_run.head_branch == '<integration>' &&
      github.event.workflow_run.head_repository.fork == false
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      # checkout the exact SHA that passed CI:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_sha }}
      # ... deploy steps for the confirmed target ...
```

Add a retry (e.g., `nick-fields/retry@v3` around the deploy step) if the target is known to be
flaky (small VPS, cold platform APIs).

## Production deploy — `deploy-<production>.yml`

Triggered by `push` to `<production>`. If the repo uses release tooling (Changesets, semantic-
release, tags), chain it: the release job runs first, and the deploy job uses
`needs: release` plus an `if:` that only deploys **when a release was actually published**
(e.g., Changesets' `steps.changesets.outputs.published == 'true'`, or `startsWith(github.ref,
'refs/tags/')` for tag-driven flows). No release tooling → deploy on every production push, with
`concurrency` NOT cancelling in-progress runs.

## Deploy step patterns per target

Pick ONE based on Phase 0 evidence; otherwise leave the dispatch-only skeleton.

- **Docker registry + VPS via SSH**: build & push to GHCR
  (`docker/build-push-action`, `push: true`, tags `ghcr.io/<owner>/<repo>:sha-<sha>` + branch
  tag), then `appleboy/ssh-action` to `docker compose pull && docker compose up -d`.
  Secrets: `SSH_HOST`, `SSH_USER`, `SSH_KEY` (private key), optionally `SSH_PORT`.
- **Fly.io**: `flyctl deploy --remote-only`. Secret: `FLY_API_TOKEN`.
- **Render**: deploy hook `curl -fsSL "$RENDER_DEPLOY_HOOK_URL"`. Secret: `RENDER_DEPLOY_HOOK_URL`.
- **Kubernetes**: build & push image, then `kubectl set image` / Helm upgrade with a kubeconfig
  secret. Secrets: `KUBECONFIG_B64` (or OIDC to the cloud provider — prefer OIDC when possible).
- **Vercel/Netlify**: usually deploy via their Git integration — in that case do NOT add a deploy
  workflow; note it in the summary instead.
- **Static/S3/CloudFront**: OIDC role + `aws s3 sync` + cache invalidation. Prefer
  `aws-actions/configure-aws-credentials` with `role-to-assume` over long-lived keys.

## Secrets policy

- Everything via `${{ secrets.NAME }}` — never a hardcoded value, never a default fallback.
- End the phase with the exact list: **secret name → where to get it → which workflow uses it**.
- Prefer OIDC federation over long-lived cloud keys wherever the target supports it.

## Checkpoint

**Guided mode**: confirm target + secret names before writing; offer the dispatch-only skeleton as
the safe alternative.
**Autopilot mode**: only deliver *active* deploys when Phase 0 found unambiguous evidence of the
target; otherwise deliver the dispatch-only skeleton and say so in the summary.
