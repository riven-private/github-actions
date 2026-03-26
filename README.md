# github-actions

CI engine, reusable workflows, and composite actions for the Riven platform.

## Architecture

```
Bot (gohan-bot)              = DISPATCHER     (webhook → check run → workflow_dispatch)
ci-engine.yml                = ORCHESTRATOR   (sequences the actions on ARC runner)
Custom composite actions     = INTELLIGENCE   (workspace detection, build, publish logic)
```

## Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `ci-engine.yml` | `workflow_dispatch` | Central CI pipeline dispatched by gohan-bot |

## Actions

| Action | Purpose |
|--------|---------|
| `setup-workspace` | Detect workspace type (Yarn/UV), setup toolchain, configure caches, CodeArtifact auth |
| `build-and-test` | Build, lint, test per workspace type with monorepo support |
| `publish-services` | Discover `riven.service` packages, docker build, ECR push |
| `report-results` | Record CI results in dev-center-service via Connect RPC |

## Usage

### Dispatched by gohan-bot (primary)

gohan-bot receives PR webhooks, creates a check run, then dispatches `ci-engine.yml` via `workflow_dispatch`. No per-repo workflow files needed.

### Direct reusable workflow call (fallback)

```yaml
# In any repo: .github/workflows/ci.yml
on: [pull_request, push]
jobs:
  ci:
    uses: riven-private/github-actions/.github/workflows/ci-engine.yml@main
    with:
      target_repo: ${{ github.repository }}
      ref: ${{ github.sha }}
```

## Self-Hosted Runners

Workflows run on `riven-runner` — ephemeral ARC runner pods on a dedicated EKS node (m5.large).
The runner has Node.js, Python, Yarn, UV, Docker (DinD), AWS CLI, and buf pre-installed.
