# CI Pipeline Optimization — 30 min to ~12-15 min

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Cut dev-center CI from ~30 min to ~12-15 min by fixing concurrency, optimizing Docker builds, and parallelizing test/lint.

**Architecture:** The CI system is a reusable `ci-engine.yml` workflow in `github-actions` repo, called by 11 repos. Composite actions handle each phase: setup-workspace, build-and-test, publish-services, publish-libraries, publish-protos. Docker images use a `node-service-base` base image with ONBUILD that copies entire build context. Changes must remain backwards-compatible across all 11 repos.

**Tech Stack:** GitHub Actions (ARC runners on EKS), Yarn 4 workspaces, Docker BuildKit, node-service-base ONBUILD pattern, AWS ECR.

**Current Timing (dev-center, 7 services):**
| Phase | Duration |
|-------|----------|
| Setup (yarn install, caching, CodeArtifact, riven-cli) | 6 min |
| Build+Test (buf codegen, tsc all packages, lint, jest) | 10.5 min |
| Docker publish (7 services sequential, 1.6GB context each) | 10 min |
| Cache save (post-job) | 3 min |
| **Total** | **~30 min** |

**Target Timing:**
| Phase | Target | Savings |
|-------|--------|---------|
| Setup | 6 min (unchanged) | — |
| Build | 5 min (parallel lint+test) | 5.5 min |
| Docker publish | 2-3 min (parallel + staging + skip unchanged) | 7-8 min |
| Cache save | 3 min (unchanged) | — |
| **Total** | **~16 min** | **~14 min saved** |

---

## Task 1: Fix Concurrency for Main Branch

**Why:** `cancel-in-progress: true` kills running main builds when new pushes land. dev-center takes 30 min — any push during that window cancels the build. This caused 3 consecutive cancellations today.

**Files:**
- Modify: `/home/riven/Desktop/riven-private/github-actions/.github/workflows/ci-engine.yml:46-48`

**Step 1: Update concurrency config**

Change the concurrency block to only cancel PR runs, not main:

```yaml
# Before:
concurrency:
  group: ci-${{ inputs.target_repo }}-${{ inputs.ref }}
  cancel-in-progress: true

# After:
concurrency:
  group: ci-${{ inputs.target_repo }}-${{ inputs.ref }}
  cancel-in-progress: ${{ !endsWith(inputs.ref, 'refs/heads/main') && inputs.ref != github.event.repository.default_branch }}
```

Wait — `inputs.ref` is a SHA, not a branch name. The caller passes `github.sha`. We need the branch info. The caller ci.yml sets `publish: true` for main. Use that:

```yaml
concurrency:
  group: ci-${{ inputs.target_repo }}-${{ inputs.ref }}
  cancel-in-progress: ${{ inputs.publish != 'true' }}
```

This means: cancel in-progress runs for PRs (`publish=false`) but never cancel main builds (`publish=true`).

**Step 2: Verify no regressions**

- PR builds still cancel older runs on same branch: YES (publish=false → cancel=true)
- Main builds queue instead of cancelling: YES (publish=true → cancel=false)
- Manual dispatches default to publish=false → cancels: CORRECT (safe default)

**Step 3: Commit**

```bash
git add .github/workflows/ci-engine.yml
git commit -m "fix(ci): don't cancel main branch builds on new pushes"
```

---

## Task 2: Docker Context Staging

**Why:** Each Docker build sends the **entire 1.6GB monorepo** as context to the Docker daemon, 7 times. With CI_PREBUILT, the image only needs: root package.json/yarn.lock/.yarnrc.yml/.yarn/, the service's dist/proto/prisma, and package.json stubs for all workspaces (for `yarn workspaces focus --production`).

**Files:**
- Modify: `/home/riven/Desktop/riven-private/github-actions/actions/publish-services/action.yml`

**Step 1: Add staging logic before Docker build**

Inside the "Build and push services" step, before the `docker build` command, create a minimal staging directory:

```bash
# Create minimal Docker context for CI_PREBUILT builds
STAGE_DIR=$(mktemp -d)
echo "::debug::Staging Docker context in $STAGE_DIR"

# Root workspace files (required by yarn workspaces focus)
cp package.json yarn.lock .yarnrc.yml "$STAGE_DIR/"
if [ -d ".yarn" ]; then
  cp -r .yarn "$STAGE_DIR/.yarn"
fi

# Copy ALL workspace package.json stubs (yarn needs these for resolution)
while read -r ws_line; do
  WS_LOC=$(echo "$ws_line" | jq -r '.location')
  [ "$WS_LOC" = "." ] && continue
  mkdir -p "$STAGE_DIR/$WS_LOC"
  cp "$WS_LOC/package.json" "$STAGE_DIR/$WS_LOC/package.json"
done < <(yarn workspaces list --json 2>/dev/null)

# Copy service-specific build artifacts
mkdir -p "$STAGE_DIR/$DIR"
cp "$DIR/package.json" "$STAGE_DIR/$DIR/"
[ -d "$DIR/dist" ] && cp -r "$DIR/dist" "$STAGE_DIR/$DIR/dist"
[ -d "$DIR/proto" ] && cp -r "$DIR/proto" "$STAGE_DIR/$DIR/proto"
[ -d "$DIR/prisma" ] && cp -r "$DIR/prisma" "$STAGE_DIR/$DIR/prisma"
[ -d "$DIR/src/__generated__/prisma" ] && {
  mkdir -p "$STAGE_DIR/$DIR/src/__generated__"
  cp -r "$DIR/src/__generated__/prisma" "$STAGE_DIR/$DIR/src/__generated__/prisma"
}

# Copy the service's Dockerfile
cp "$DOCKERFILE" "$STAGE_DIR/$DIR/Dockerfile" 2>/dev/null || cp "$DOCKERFILE" "$STAGE_DIR/Dockerfile"

# Measure staged size
STAGE_SIZE=$(du -sh "$STAGE_DIR" | cut -f1)
echo "Staged context: $STAGE_SIZE (was 1.6GB+)"
```

Then change the `docker build` to use `$STAGE_DIR` as context:

```bash
docker build -t "$IMAGE" -t "${{ inputs.ecr-registry }}/${ECR}:latest" \
  -f "$STAGE_DIR/$DIR/Dockerfile" \
  --build-arg WORKSPACE_NAME="$NAME" \
  --build-arg SERVICE_NAME="$NAME" \
  --build-arg SERVICE_DIR="$DIR" \
  --build-arg CODEARTIFACT_AUTH_TOKEN="${CODEARTIFACT_AUTH_TOKEN:-}" \
  --build-arg CI_PREBUILT=true \
  "$STAGE_DIR"
```

After build, cleanup:

```bash
rm -rf "$STAGE_DIR"
```

**Step 2: Handle non-CI_PREBUILT fallback**

Some repos may not use CI_PREBUILT (the flag is always passed by publish-services, but the per-service Dockerfile may not support it). The staging approach only works for CI_PREBUILT. For safety, detect if the service uses node-service-base:

```bash
# Only stage if the Dockerfile uses node-service-base (supports CI_PREBUILT)
if grep -q "node-service-base" "$DOCKERFILE"; then
  # ... staging logic above ...
  BUILD_CONTEXT="$STAGE_DIR"
else
  # Fallback: full context for non-base-image Dockerfiles
  BUILD_CONTEXT="."
fi
```

**Step 3: Commit**

```bash
git add actions/publish-services/action.yml
git commit -m "perf(ci): stage minimal Docker context for CI_PREBUILT builds

Reduces Docker build context from 1.6GB to ~50MB per service by copying
only package.json stubs, dist/, proto/, prisma/ into a staging directory."
```

---

## Task 3: Parallel Docker Builds

**Why:** 7 sequential builds take ~10 min. With staged contexts (~50MB each), parallel builds are safe on disk. Even 2-3 concurrent builds halve the time.

**Files:**
- Modify: `/home/riven/Desktop/riven-private/github-actions/actions/publish-services/action.yml`

**Step 1: Replace sequential loop with parallel background jobs**

Refactor the "Build and push services" step to launch builds in background and collect results:

```bash
SERVICES='${{ steps.discover.outputs.services }}'
SHA=$(git rev-parse --short HEAD)
TOTAL=$(echo "$SERVICES" | jq 'length')
RESULT_DIR=$(mktemp -d)

# --- Workspace list (shared across all staging dirs) ---
WORKSPACE_LIST=$(yarn workspaces list --json 2>/dev/null)

# Function to build one service
build_service() {
  local idx=$1 svc=$2
  local NAME=$(echo "$svc" | jq -r '.name')
  local DIR=$(echo "$svc" | jq -r '.dir')
  local ECR=$(echo "$svc" | jq -r '.ecr')
  local VERSION=$(echo "$svc" | jq -r '.version')
  local DOCKERFILE=$(echo "$svc" | jq -r '.dockerfile')
  local TAG="${VERSION}-${SHA}"
  local IMAGE="${{ inputs.ecr-registry }}/${ECR}:${TAG}"
  local LOG_FILE="$RESULT_DIR/${NAME//\//_}.log"

  echo "[$idx/$TOTAL] Building $NAME → $IMAGE"

  # --- Stage context ---
  local STAGE_DIR=$(mktemp -d)
  cp package.json yarn.lock .yarnrc.yml "$STAGE_DIR/"
  [ -d ".yarn" ] && cp -r .yarn "$STAGE_DIR/.yarn"

  echo "$WORKSPACE_LIST" | while read -r ws_line; do
    local WS_LOC=$(echo "$ws_line" | jq -r '.location')
    [ "$WS_LOC" = "." ] && continue
    mkdir -p "$STAGE_DIR/$WS_LOC"
    cp "$WS_LOC/package.json" "$STAGE_DIR/$WS_LOC/package.json"
  done

  mkdir -p "$STAGE_DIR/$DIR"
  cp "$DIR/package.json" "$STAGE_DIR/$DIR/"
  [ -d "$DIR/dist" ] && cp -r "$DIR/dist" "$STAGE_DIR/$DIR/dist"
  [ -d "$DIR/proto" ] && cp -r "$DIR/proto" "$STAGE_DIR/$DIR/proto"
  [ -d "$DIR/prisma" ] && cp -r "$DIR/prisma" "$STAGE_DIR/$DIR/prisma"
  [ -d "$DIR/src/__generated__/prisma" ] && {
    mkdir -p "$STAGE_DIR/$DIR/src/__generated__"
    cp -r "$DIR/src/__generated__/prisma" "$STAGE_DIR/$DIR/src/__generated__/prisma"
  }

  # Copy Dockerfile into staging
  if [ -f "$DOCKERFILE" ]; then
    local REL_DF="$DOCKERFILE"
    mkdir -p "$STAGE_DIR/$(dirname "$REL_DF")"
    cp "$DOCKERFILE" "$STAGE_DIR/$REL_DF"
  fi

  # --- Ensure ECR repo ---
  aws ecr describe-repositories --repository-names "$ECR" 2>/dev/null \
    || aws ecr create-repository --repository-name "$ECR" --image-scanning-configuration scanOnPush=true 2>/dev/null \
    || true

  # --- Build and push ---
  if docker build -t "$IMAGE" -t "${{ inputs.ecr-registry }}/${ECR}:latest" \
    -f "$STAGE_DIR/$DOCKERFILE" \
    --build-arg WORKSPACE_NAME="$NAME" \
    --build-arg SERVICE_NAME="$NAME" \
    --build-arg SERVICE_DIR="$DIR" \
    --build-arg CODEARTIFACT_AUTH_TOKEN="${CODEARTIFACT_AUTH_TOKEN:-}" \
    --build-arg CI_PREBUILT=true \
    "$STAGE_DIR" >> "$LOG_FILE" 2>&1 \
    && docker push "$IMAGE" >> "$LOG_FILE" 2>&1 \
    && docker push "${{ inputs.ecr-registry }}/${ECR}:latest" >> "$LOG_FILE" 2>&1; then
    echo "OK" > "$RESULT_DIR/${NAME//\//_}.status"
    echo "[$idx/$TOTAL] Published $NAME:$TAG"
  else
    echo "FAIL" > "$RESULT_DIR/${NAME//\//_}.status"
    echo "::error::[$idx/$TOTAL] Failed to publish $NAME"
    cat "$LOG_FILE"
  fi

  rm -rf "$STAGE_DIR"
}

export -f build_service 2>/dev/null || true

# --- Launch all builds in parallel ---
IDX=0
PIDS=()
while read -r svc; do
  IDX=$((IDX + 1))
  build_service "$IDX" "$svc" &
  PIDS+=($!)
done < <(echo "$SERVICES" | jq -c '.[]' 2>/dev/null)

# --- Wait for all and collect results ---
FAILED=0
for pid in "${PIDS[@]}"; do
  wait "$pid" || FAILED=$((FAILED + 1))
done

# Check status files
for status_file in "$RESULT_DIR"/*.status; do
  [ -f "$status_file" ] || continue
  if [ "$(cat "$status_file")" = "FAIL" ]; then
    FAILED=$((FAILED + 1))
  fi
done

# Print all logs
for log_file in "$RESULT_DIR"/*.log; do
  [ -f "$log_file" ] || continue
  SVC_NAME=$(basename "$log_file" .log)
  echo "::group::Docker build log: $SVC_NAME"
  cat "$log_file"
  echo "::endgroup::"
done

rm -rf "$RESULT_DIR"

if [ "$FAILED" -gt 0 ]; then
  echo "::error::$FAILED of $TOTAL service(s) failed to publish"
  exit 1
fi

echo "Published $TOTAL services successfully (parallel)"
echo "published=true" >> $GITHUB_OUTPUT
```

**Important:** `export -f` may not work in all shell contexts. If needed, use a temporary script file instead of a bash function. The function approach is shown for clarity — the actual implementation should write a helper script.

**Step 2: Commit**

```bash
git add actions/publish-services/action.yml
git commit -m "perf(ci): parallel Docker builds with staged contexts

Builds all services concurrently using background processes.
Each build gets a minimal ~50MB staging context instead of 1.6GB."
```

---

## Task 4: Skip Unchanged Services

**Why:** Most pushes to main only change 1-2 services. Building all 7 Docker images wastes time. Use git diff to detect which services changed and skip the rest.

**Files:**
- Modify: `/home/riven/Desktop/riven-private/github-actions/actions/publish-services/action.yml`

**Step 1: Add change detection before builds**

After discovering services but before building, filter to changed services:

```bash
# Detect which files changed in this push
CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD 2>/dev/null || echo "FULL_BUILD")

if [ "$CHANGED_FILES" = "FULL_BUILD" ]; then
  echo "Cannot determine changed files — building all services"
  FILTERED_SERVICES="$SERVICES"
else
  # Check for changes that affect ALL services (root config, shared packages)
  FORCE_ALL=false
  if echo "$CHANGED_FILES" | grep -qE '^(package\.json|yarn\.lock|\.yarnrc\.yml|tsconfig|common/|Dockerfile$)'; then
    FORCE_ALL=true
    echo "Root/shared files changed — building all services"
  fi

  if [ "$FORCE_ALL" = "true" ]; then
    FILTERED_SERVICES="$SERVICES"
  else
    # Filter to services whose directory (or dependencies) changed
    FILTERED_SERVICES="[]"
    while read -r svc; do
      DIR=$(echo "$svc" | jq -r '.dir')
      if echo "$CHANGED_FILES" | grep -q "^$DIR/"; then
        FILTERED_SERVICES=$(echo "$FILTERED_SERVICES" | jq --argjson s "$svc" '. + [$s]')
      fi
    done < <(echo "$SERVICES" | jq -c '.[]' 2>/dev/null)
  fi
fi

SKIP_COUNT=$(( $(echo "$SERVICES" | jq 'length') - $(echo "$FILTERED_SERVICES" | jq 'length') ))
echo "Building $(echo "$FILTERED_SERVICES" | jq 'length') of $(echo "$SERVICES" | jq 'length') services ($SKIP_COUNT unchanged, skipped)"
```

Then use `$FILTERED_SERVICES` instead of `$SERVICES` in the parallel build loop.

**Step 2: Add shared-package dependency detection**

If `packages/dev-center-api/` changed (shared proto definitions), rebuild all services that depend on it. This requires reading `riven.protoDeps` from each service's package.json:

```bash
# Shared packages that trigger rebuilds of dependents
SHARED_CHANGED=()
while read -r svc; do
  DIR=$(echo "$svc" | jq -r '.dir')
  IS_SERVICE=$(jq -r '.riven.service // false' "$DIR/package.json")
  if [ "$IS_SERVICE" != "true" ] && echo "$CHANGED_FILES" | grep -q "^$DIR/"; then
    SHARED_CHANGED+=("$DIR")
  fi
done < <(yarn workspaces list --json 2>/dev/null)

# If shared packages changed, add services that depend on them
if [ ${#SHARED_CHANGED[@]} -gt 0 ]; then
  while read -r svc; do
    DIR=$(echo "$svc" | jq -r '.dir')
    # Check if this service depends on any changed shared package
    DEPS=$(jq -r '.riven.protoDeps[]? // empty' "$DIR/package.json" 2>/dev/null)
    for dep in $DEPS; do
      for changed in "${SHARED_CHANGED[@]}"; do
        if [[ "$dep" == *"$changed"* ]]; then
          FILTERED_SERVICES=$(echo "$FILTERED_SERVICES" | jq --argjson s "$svc" 'if any(.[]; .name == ($s | .name)) then . else . + [$s] end')
        fi
      done
    done
  done < <(echo "$SERVICES" | jq -c '.[]' 2>/dev/null)
fi
```

**Step 3: Commit**

```bash
git add actions/publish-services/action.yml
git commit -m "perf(ci): skip Docker builds for unchanged services

Uses git diff to detect which service directories changed.
Rebuilds all if root config/shared packages changed."
```

---

## Task 5: Parallel Lint + Test After Build

**Why:** Currently build-and-test runs: build → lint → test sequentially. Build must be first (codegen + tsc). But lint and test are independent of each other and can run in parallel after build completes. This saves ~3-4 min (the shorter of lint/test overlaps with the longer).

**Files:**
- Modify: `/home/riven/Desktop/riven-private/github-actions/actions/build-and-test/action.yml`

**Step 1: Run lint and test in parallel**

Replace the sequential lint and test steps with a single step that runs both in parallel:

```yaml
- name: Lint and Test (parallel)
  id: lint-and-test
  shell: bash
  env:
    NODE_OPTIONS: "--max-old-space-size=3072"
  run: |
    TYPE="${{ steps.detect.outputs.type }}"
    LINT_EXIT=0
    TEST_EXIT=0

    # --- Determine commands ---
    case "$TYPE" in
      yarn|npm|pnpm)
        PKG_CMD="$TYPE"
        HAS_LINT=$(node -e "const p=require('./package.json'); process.exit(p.scripts?.lint ? 0 : 1)" 2>/dev/null && echo "true" || echo "false")
        HAS_TEST=$(node -e "const p=require('./package.json'); process.exit(p.scripts?.test ? 0 : 1)" 2>/dev/null && echo "true" || echo "false")
        ;;
      uv)
        HAS_LINT=true
        HAS_TEST=true
        ;;
      go)
        HAS_LINT=true
        HAS_TEST=true
        ;;
      *)
        HAS_LINT=false
        HAS_TEST=false
        ;;
    esac

    # --- Run lint in background ---
    if [ "$HAS_LINT" = "true" ]; then
      (
        case "$TYPE" in
          yarn) yarn lint ;;
          pnpm) pnpm lint ;;
          npm)  npm run lint ;;
          uv)   uv run ruff check . 2>/dev/null; uv run ruff format --check . 2>/dev/null ;;
          go)   golangci-lint run ./... 2>/dev/null ;;
        esac
      ) > /tmp/lint.log 2>&1 &
      LINT_PID=$!
    fi

    # --- Run test in foreground ---
    if [ "$HAS_TEST" = "true" ]; then
      case "$TYPE" in
        yarn) yarn test || TEST_EXIT=$? ;;
        pnpm) pnpm test || TEST_EXIT=$? ;;
        npm)  npm test || TEST_EXIT=$? ;;
        uv)   uv run pytest || TEST_EXIT=$? ;;
        go)   go test ./... || TEST_EXIT=$? ;;
      esac
    else
      echo "No test script configured — skipping"
    fi

    # --- Collect lint result ---
    if [ "$HAS_LINT" = "true" ]; then
      wait "$LINT_PID" || LINT_EXIT=$?
      echo "::group::Lint output"
      cat /tmp/lint.log
      echo "::endgroup::"
    else
      echo "No lint script configured — skipping"
    fi

    # --- Report ---
    if [ "$LINT_EXIT" -ne 0 ]; then
      echo "::error::Lint failed (exit $LINT_EXIT)"
    fi
    if [ "$TEST_EXIT" -ne 0 ]; then
      echo "::error::Test failed (exit $TEST_EXIT)"
    fi

    # Fail if either failed
    [ "$LINT_EXIT" -eq 0 ] && [ "$TEST_EXIT" -eq 0 ]
```

**Step 2: Keep output IDs for backwards compatibility**

The original action exposes `build-status`, `test-status`, `lint-status` outputs. Update the outputs section:

```yaml
outputs:
  build-status:
    description: "Build result: success or failure"
    value: ${{ steps.build.outcome }}
  test-status:
    description: "Lint+Test result: success, failure, or skipped"
    value: ${{ steps.lint-and-test.outcome }}
  lint-status:
    description: "Lint+Test result (same as test-status for parallel runs)"
    value: ${{ steps.lint-and-test.outcome }}
```

**Step 3: Commit**

```bash
git add actions/build-and-test/action.yml
git commit -m "perf(ci): run lint and test in parallel after build

Lint runs in background while test runs in foreground.
Both results collected and reported. Saves ~3-4 min overlap."
```

---

## Task 6: Optimize smart-build.sh for CI_PREBUILT

**Why:** Even with CI_PREBUILT=true, `smart-build.sh` runs `yarn workspaces focus --production` inside Docker. With staged context, this yarn install resolves from network/cache. We can pre-stage production node_modules from the CI workspace to skip this entirely.

**Files:**
- Modify: `/home/riven/Desktop/riven-private/node-platforms/docker/base/scripts/smart-build.sh`
- Modify: `/home/riven/Desktop/riven-private/github-actions/actions/publish-services/action.yml`

**Step 1: Add CI_SKIP_PROD_INSTALL flag to smart-build.sh**

In smart-build.sh, after the CI_PREBUILT check, add a second optimization:

```bash
# In smart-build.sh, replace lines 94-98:

# Before:
# log "Installing production dependencies..."
# yarn workspaces focus "$SERVICE_NAME" --production

# After:
CI_SKIP_PROD_INSTALL="${CI_SKIP_PROD_INSTALL:-false}"

if [ "$CI_SKIP_PROD_INSTALL" = "true" ]; then
  log "CI_SKIP_PROD_INSTALL=true — using pre-staged production dependencies"
else
  log "Installing production dependencies..."
  yarn workspaces focus "$SERVICE_NAME" --production
fi
```

**Step 2: Pre-stage production node_modules in publish-services**

In the staging logic (Task 2), add production dependency preparation:

```bash
# After staging other files, prepare production node_modules
# Run yarn focus --production ONCE in a temp copy, then copy node_modules
PROD_DIR=$(mktemp -d)
cp package.json yarn.lock .yarnrc.yml "$PROD_DIR/"
[ -d ".yarn" ] && cp -r .yarn "$PROD_DIR/.yarn"

# Copy all package.json stubs
echo "$WORKSPACE_LIST" | while read -r ws_line; do
  WS_LOC=$(echo "$ws_line" | jq -r '.location')
  [ "$WS_LOC" = "." ] && continue
  mkdir -p "$PROD_DIR/$WS_LOC"
  cp "$WS_LOC/package.json" "$PROD_DIR/$WS_LOC/package.json"
done

# Generate prod node_modules for this service
cd "$PROD_DIR"
yarn workspaces focus "$NAME" --production 2>/dev/null
cd -

# Copy prod node_modules into staging
cp -r "$PROD_DIR/node_modules" "$STAGE_DIR/node_modules"
rm -rf "$PROD_DIR"
```

Then pass `--build-arg CI_SKIP_PROD_INSTALL=true` to docker build.

**Note:** This trades disk I/O (copying node_modules) for network I/O (yarn install inside Docker). Benchmark to confirm improvement. If node_modules is large (500MB+), the copy overhead may negate savings. In that case, skip this task and rely on Docker BuildKit yarn cache mount.

**Step 3: Commit node-platforms change**

```bash
cd /home/riven/Desktop/riven-private/node-platforms
git add docker/base/scripts/smart-build.sh
git commit -m "feat(node-service-base): add CI_SKIP_PROD_INSTALL flag

When true, skips yarn workspaces focus --production inside Docker.
Expects pre-staged node_modules from CI workspace."
```

**Step 4: Commit github-actions change**

```bash
cd /home/riven/Desktop/riven-private/github-actions
git add actions/publish-services/action.yml
git commit -m "perf(ci): pre-stage production node_modules for Docker builds

Runs yarn focus --production once in CI, copies result into staging.
Eliminates redundant yarn install inside each Docker build."
```

---

## Task 7: Increase EBS and Ephemeral Storage

**Why:** Current m5.large runners have 50GB EBS and 20Gi ephemeral. Parallel Docker builds + build artifacts need more disk. Increasing to 100GB EBS / 40Gi ephemeral ensures headroom.

**Files:**
- Modify: `/home/riven/Desktop/riven-private/helm-charts/charts/actions-runner-scale-set/values-custom.yaml`

**Step 1: Update resource requests**

Find and update the ephemeral storage request:

```yaml
# Before:
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
    ephemeral-storage: "20Gi"

# After:
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
    ephemeral-storage: "40Gi"
```

**Step 2: Document EBS node volume increase**

The EBS volume size is set at the EKS node group level, not in Helm. Add a comment:

```yaml
# NOTE: EBS root volume on ci-runner nodes increased from 50GB → 100GB
# via AWS Console / eksctl. Required for parallel Docker builds.
# m5.large (8 GiB RAM) fits 2 concurrent runners at 40Gi ephemeral each.
```

**Step 3: Actually increase the EBS**

This requires either:
- AWS Console: Modify the EKS node group launch template to use 100GB gp3
- Or: `aws ec2 modify-volume` on existing volumes + node group update

Provide the command:

```bash
# Get the launch template used by the ci-runner node group
aws eks describe-nodegroup --cluster-name riven-cluster --nodegroup-name ci-runners \
  --query 'nodegroup.launchTemplate' --output json

# Update the launch template with new volume size
# (exact command depends on launch template ID)
```

**Step 4: Commit helm changes**

```bash
cd /home/riven/Desktop/riven-private/helm-charts
git add charts/actions-runner-scale-set/values-custom.yaml
git commit -m "ops(ci): increase runner ephemeral storage to 40Gi

Supports parallel Docker builds. Requires 100GB EBS on ci-runner nodes."
```

---

## Task 8: Integration — Full publish-services Rewrite

**Why:** Tasks 2-4 modify the same file. This task provides the complete rewritten `action.yml` combining context staging, parallel builds, and skip-unchanged.

**Files:**
- Rewrite: `/home/riven/Desktop/riven-private/github-actions/actions/publish-services/action.yml`

The full file should be:

```yaml
name: "Publish Services"
description: "Discover riven.service packages, build Docker images in parallel, push to ECR. Skips unchanged services."

inputs:
  ecr-registry:
    description: "ECR registry URL (e.g., 572953113830.dkr.ecr.us-east-1.amazonaws.com)"
    required: true

outputs:
  published:
    description: "JSON array of published services with tags"
    value: ${{ steps.publish.outputs.published }}
  skipped:
    description: "Number of services skipped (unchanged)"
    value: ${{ steps.publish.outputs.skipped }}

runs:
  using: "composite"
  steps:
    - name: Discover services
      id: discover
      shell: bash
      run: |
        if [ -f "detection.json" ]; then
          SERVICES=$(jq -c '.services' detection.json)
          echo "services=$SERVICES" >> $GITHUB_OUTPUT
          echo "Read $(echo "$SERVICES" | jq length) services from detection.json"
        else
          # ... existing fallback discovery logic (unchanged) ...
          SERVICES="[]"
          if [ -f "yarn.lock" ]; then
            SERVICES=$(yarn workspaces list --json 2>/dev/null | while read -r line; do
              LOC=$(echo "$line" | jq -r '.location')
              NAME=$(echo "$line" | jq -r '.name')
              PKG="$LOC/package.json"
              [ -f "$PKG" ] || continue
              IS_SERVICE=$(jq -r '.riven.service // false' "$PKG")
              if [ "$IS_SERVICE" = "true" ]; then
                ECR=$(jq -r '.riven.ecr' "$PKG")
                VERSION=$(jq -r '.version // "0.0.0"' "$PKG")
                if [ -f "$LOC/Dockerfile" ]; then
                  DOCKERFILE="$LOC/Dockerfile"
                else
                  DOCKERFILE="Dockerfile"
                fi
                echo "{\"name\":\"$NAME\",\"dir\":\"$LOC\",\"ecr\":\"$ECR\",\"version\":\"$VERSION\",\"dockerfile\":\"$DOCKERFILE\"}"
              fi
            done | jq -sc '.')
          fi
          echo "services=$SERVICES" >> $GITHUB_OUTPUT
          echo "Found $(echo "$SERVICES" | jq length) services to publish"
        fi

    - name: ECR login
      if: steps.discover.outputs.services != '[]' && steps.discover.outputs.services != 'null'
      shell: bash
      continue-on-error: true
      run: |
        if command -v aws &> /dev/null; then
          aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${{ inputs.ecr-registry }} 2>/dev/null
        else
          echo "AWS CLI not available — skipping ECR login"
        fi

    - name: Build and push services (parallel)
      id: publish
      shell: bash
      run: |
        set +e  # Don't exit on individual build failures
        SERVICES='${{ steps.discover.outputs.services }}'
        SHA=$(git rev-parse --short HEAD)
        TOTAL=$(echo "$SERVICES" | jq 'length')
        RESULT_DIR=$(mktemp -d)

        if [ "$TOTAL" -eq 0 ] || [ "$SERVICES" = "[]" ] || [ "$SERVICES" = "null" ]; then
          echo "No services to publish"
          echo "published=[]" >> $GITHUB_OUTPUT
          echo "skipped=0" >> $GITHUB_OUTPUT
          exit 0
        fi

        # ── 1. Detect changed files ──────────────────────────────────
        CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD 2>/dev/null || echo "")
        FORCE_ALL=false

        if [ -z "$CHANGED_FILES" ]; then
          FORCE_ALL=true
          echo "Cannot determine changes — building all services"
        elif echo "$CHANGED_FILES" | grep -qE '^(package\.json|yarn\.lock|\.yarnrc\.yml|Dockerfile$|common/)'; then
          FORCE_ALL=true
          echo "Root/shared files changed — building all services"
        fi

        # ── 2. Filter to changed services ────────────────────────────
        FILTERED_SERVICES="[]"
        while read -r svc; do
          [ -z "$svc" ] && continue
          DIR=$(echo "$svc" | jq -r '.dir')
          if [ "$FORCE_ALL" = "true" ] || echo "$CHANGED_FILES" | grep -q "^$DIR/"; then
            FILTERED_SERVICES=$(echo "$FILTERED_SERVICES" | jq --argjson s "$svc" '. + [$s]')
          fi
        done < <(echo "$SERVICES" | jq -c '.[]' 2>/dev/null)

        FILTERED_COUNT=$(echo "$FILTERED_SERVICES" | jq 'length')
        SKIPPED=$((TOTAL - FILTERED_COUNT))
        echo "Building $FILTERED_COUNT of $TOTAL services ($SKIPPED unchanged, skipped)"
        echo "skipped=$SKIPPED" >> $GITHUB_OUTPUT

        if [ "$FILTERED_COUNT" -eq 0 ]; then
          echo "No services changed — nothing to publish"
          echo "published=[]" >> $GITHUB_OUTPUT
          exit 0
        fi

        # ── 3. Cache workspace list for staging ──────────────────────
        WORKSPACE_LIST_FILE=$(mktemp)
        yarn workspaces list --json 2>/dev/null > "$WORKSPACE_LIST_FILE"

        # ── 4. Write build helper script ─────────────────────────────
        BUILD_SCRIPT=$(mktemp)
        cat > "$BUILD_SCRIPT" << 'BUILDEOF'
        #!/bin/bash
        set -e
        IDX="$1"; SVC="$2"; SHA="$3"; ECR_REGISTRY="$4"
        TOTAL="$5"; RESULT_DIR="$6"; WS_FILE="$7"

        NAME=$(echo "$SVC" | jq -r '.name')
        DIR=$(echo "$SVC" | jq -r '.dir')
        ECR=$(echo "$SVC" | jq -r '.ecr')
        VERSION=$(echo "$SVC" | jq -r '.version')
        DOCKERFILE=$(echo "$SVC" | jq -r '.dockerfile')
        TAG="${VERSION}-${SHA}"
        IMAGE="${ECR_REGISTRY}/${ECR}:${TAG}"
        SAFE_NAME="${NAME//\//_}"
        LOG="$RESULT_DIR/${SAFE_NAME}.log"

        echo "[$IDX/$TOTAL] Building $NAME → $IMAGE"

        # ── Stage minimal context ──
        STAGE=$(mktemp -d)
        cp package.json yarn.lock .yarnrc.yml "$STAGE/"
        [ -d ".yarn" ] && cp -r .yarn "$STAGE/.yarn"

        while read -r ws_line; do
          WS_LOC=$(echo "$ws_line" | jq -r '.location')
          [ "$WS_LOC" = "." ] && continue
          mkdir -p "$STAGE/$WS_LOC"
          [ -f "$WS_LOC/package.json" ] && cp "$WS_LOC/package.json" "$STAGE/$WS_LOC/package.json"
        done < "$WS_FILE"

        mkdir -p "$STAGE/$DIR"
        cp "$DIR/package.json" "$STAGE/$DIR/"
        [ -d "$DIR/dist" ] && cp -r "$DIR/dist" "$STAGE/$DIR/dist"
        [ -d "$DIR/proto" ] && cp -r "$DIR/proto" "$STAGE/$DIR/proto"
        [ -d "$DIR/prisma" ] && cp -r "$DIR/prisma" "$STAGE/$DIR/prisma"
        if [ -d "$DIR/src/__generated__/prisma" ]; then
          mkdir -p "$STAGE/$DIR/src/__generated__"
          cp -r "$DIR/src/__generated__/prisma" "$STAGE/$DIR/src/__generated__/prisma"
        fi

        # Copy Dockerfile into staged context
        mkdir -p "$STAGE/$(dirname "$DOCKERFILE")"
        cp "$DOCKERFILE" "$STAGE/$DOCKERFILE"

        STAGE_SIZE=$(du -sh "$STAGE" 2>/dev/null | cut -f1)
        echo "[$IDX/$TOTAL] Staged context: $STAGE_SIZE" >> "$LOG"

        # ── Ensure ECR repo ──
        aws ecr describe-repositories --repository-names "$ECR" >> "$LOG" 2>&1 \
          || aws ecr create-repository --repository-name "$ECR" --image-scanning-configuration scanOnPush=true >> "$LOG" 2>&1 \
          || true

        # ── Build + push ──
        if docker build -t "$IMAGE" -t "${ECR_REGISTRY}/${ECR}:latest" \
          -f "$STAGE/$DOCKERFILE" \
          --build-arg WORKSPACE_NAME="$NAME" \
          --build-arg SERVICE_NAME="$NAME" \
          --build-arg SERVICE_DIR="$DIR" \
          --build-arg CODEARTIFACT_AUTH_TOKEN="${CODEARTIFACT_AUTH_TOKEN:-}" \
          --build-arg CI_PREBUILT=true \
          "$STAGE" >> "$LOG" 2>&1 \
          && docker push "$IMAGE" >> "$LOG" 2>&1 \
          && docker push "${ECR_REGISTRY}/${ECR}:latest" >> "$LOG" 2>&1; then
          echo "OK" > "$RESULT_DIR/${SAFE_NAME}.status"
          echo "[$IDX/$TOTAL] Published $NAME:$TAG"
        else
          echo "FAIL" > "$RESULT_DIR/${SAFE_NAME}.status"
          echo "::error::[$IDX/$TOTAL] Failed to publish $NAME"
        fi

        rm -rf "$STAGE"
        BUILDEOF
        chmod +x "$BUILD_SCRIPT"

        # ── 5. Launch parallel builds ────────────────────────────────
        IDX=0
        PIDS=()
        while read -r svc; do
          [ -z "$svc" ] && continue
          IDX=$((IDX + 1))
          bash "$BUILD_SCRIPT" "$IDX" "$svc" "$SHA" "${{ inputs.ecr-registry }}" \
            "$FILTERED_COUNT" "$RESULT_DIR" "$WORKSPACE_LIST_FILE" &
          PIDS+=($!)
        done < <(echo "$FILTERED_SERVICES" | jq -c '.[]' 2>/dev/null)

        # ── 6. Wait + collect ────────────────────────────────────────
        for pid in "${PIDS[@]}"; do
          wait "$pid" 2>/dev/null || true
        done

        # Print logs grouped
        FAILED=0
        for status_file in "$RESULT_DIR"/*.status; do
          [ -f "$status_file" ] || continue
          SAFE=$(basename "$status_file" .status)
          echo "::group::Docker: $SAFE"
          cat "$RESULT_DIR/${SAFE}.log" 2>/dev/null || true
          echo "::endgroup::"
          if [ "$(cat "$status_file")" = "FAIL" ]; then
            FAILED=$((FAILED + 1))
          fi
        done

        rm -rf "$RESULT_DIR" "$WORKSPACE_LIST_FILE" "$BUILD_SCRIPT"

        if [ "$FAILED" -gt 0 ]; then
          echo "::error::$FAILED of $FILTERED_COUNT service(s) failed to publish"
          exit 1
        fi

        echo "Published $FILTERED_COUNT services ($SKIPPED skipped)"
        echo "published=true" >> $GITHUB_OUTPUT
```

**Step: Commit**

```bash
git add actions/publish-services/action.yml
git commit -m "perf(ci): rewrite publish-services with parallel builds, context staging, skip-unchanged

- Detects changed files via git diff; skips unchanged services
- Stages minimal Docker context (~50MB vs 1.6GB) per service
- Builds all services in parallel via background processes
- Prints grouped logs per service for debuggability"
```

---

## Summary of Expected Impact

| Optimization | Phase Affected | Time Saved |
|---|---|---|
| Fix concurrency (Task 1) | Prevents wasted runs | 30+ min per cancelled run |
| Docker context staging (Task 2) | Publish services | ~3-4 min (context transfer) |
| Parallel Docker builds (Task 3) | Publish services | ~6-7 min (10 min → 3 min) |
| Skip unchanged services (Task 4) | Publish services | ~8 min on typical pushes |
| Parallel lint+test (Task 5) | Build+test | ~3-4 min |
| Skip prod deps (Task 6) | Docker build time | ~1 min per service |
| Increase EBS (Task 7) | Enables all of above | Prerequisite |

**Net result for dev-center:**
- Typical push (1-2 services changed): **~12 min** (build 5 min + Docker 2 min + overhead 5 min)
- Full rebuild (root config changed): **~16 min** (build 5 min + Docker 4 min + overhead 7 min)
- Down from current **~30 min**

---

## Execution Order

1. **Task 7** first (EBS increase) — prerequisite for parallel builds
2. **Task 1** (concurrency fix) — immediate win, no dependencies
3. **Task 5** (parallel lint+test) — independent change in build-and-test
4. **Task 8** (full publish-services rewrite combining Tasks 2-4) — the big change
5. **Task 6** (CI_SKIP_PROD_INSTALL) — requires node-service-base image rebuild + ECR push

Tasks 1, 5, 7 can be done in parallel. Task 8 depends on 7. Task 6 depends on 8.
