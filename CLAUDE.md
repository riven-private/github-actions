# CLAUDE.md — github-actions

PUBLIC repo containing reusable GitHub Actions workflows and composite actions for CI/CD.

## Layout

```
.github/workflows/
  ci-engine.yml          # Central CI workflow (workflow_dispatch + workflow_call)

actions/
  setup-workspace/       # Detect repo type, install deps, CodeArtifact auth, output detection.json
  build-and-test/        # Codegen, build, lint, test (reads detection.json)
  publish-services/      # Docker build + ECR push (reads detection.json)
  publish-libraries/     # JS -> CodeArtifact npm, Python -> CodeArtifact PyPI
  publish-protos/        # buf -> proto-registry
  report-results/        # Record results in dev-center via Connect RPC
  shared/
    defaults.env         # Centralized CI config (AWS region, CodeArtifact, ECR registry)
```

## How It Works

1. `gohan-bot-app` receives PR webhook -> dispatches `ci-engine.yml` via `workflow_dispatch`
2. `setup-workspace` detects repo type (JS/Python), installs deps, outputs `detection.json`
3. Subsequent actions read `detection.json` to avoid re-detecting
4. On publish=true: Docker images pushed to ECR, libraries to CodeArtifact, protos to registry

## Conventions

- This is a PUBLIC repo — never commit secrets or internal URLs
- All repos reference this via `riven-private/github-actions/.github/workflows/ci-engine.yml@main`
- Publish steps are gated on `inputs.publish == 'true'`
