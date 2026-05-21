# muxly-ci

Reusable GitHub Actions workflows for Muxly services.

## Versioning

Reference workflows by the `@v1` sliding tag:

```yaml
uses: Muxly-Corp/muxly-ci/.github/workflows/publish-go.yml@v1
```

`v1` always points to the latest stable commit on `main`. Breaking changes get a new tag (`v2`); migration is service-by-service.

## Workflows

### `publish-go.yml`

Build, test, optionally check generated code, optionally warn on stale BSR schema SDKs, push Docker image to ghcr, and (on push to main) clear Keel pause in dev clusters.

| Input | Type | Default | Description |
|---|---|---|---|
| `image_name` | string | required | `ghcr.io/muxly-corp/<image_name>:latest` is the published tag. |
| `warn_stale_bsr_schemas` | bool | `true` | Advisory check comparing pinned `buf.build/gen/muxly/schemas/*` versions in `go.mod` against latest BSR. Fails the run but does **not** block `publish`. |
| `check_di_generation` | bool | `false` | Run `make generate` and `git diff --exit-code -- internal/di/easydi_gen.go`. Only services using easydi. |
| `clear_keel_pause` | bool | `true` | Run `clear-keel-pause` job per cluster after successful publish. Disable for services without a long-running Deployment. |
| `deployment_name` | string | `image_name` | k8s Deployment name. Override when repo / image name ≠ Deployment name. |
| `clusters` | string | `'["dev-ru","dev-eu"]'` | JSON array of cluster suffixes for `clear-keel-pause` matrix. Prod intentionally excluded by default. |
| `go_version_file` | string | `go.mod` | Passed to `actions/setup-go`. |

Secrets (via `secrets: inherit`):
- `GH_PAT` (per-repo on GitHub Free; org-level on Team+) — for fetching private Go modules.
- `GITHUB_TOKEN` (auto) — for ghcr push and kubectl on in-cluster runners.

**Caller must grant permissions:** the `publish` job pushes to ghcr, so each wrapper workflow must declare

```yaml
permissions:
  contents: read
  packages: write
```

at the workflow root. GitHub fails the called workflow at startup (`startup_failure`, no jobs, no annotations) if the caller does not match the called job's permission grants.

### `rollback-k8s.yml`

Pause Keel on a deployment, `kubectl rollout undo`, log before/after image and revision.

| Input | Type | Required | Description |
|---|---|---|---|
| `cluster` | string | yes | `dev-ru`, `dev-eu`, `prod-ru`, or `prod-eu`. |
| `deployment_name` | string | yes | k8s Deployment name. |
| `timeout_seconds` | number | no (120) | `kubectl rollout status --timeout`. |
| `to_revision` | string | no (empty) | If set, `kubectl rollout undo --to-revision=N`. |

Namespace mapping: `dev-*` → `muxly-dev`, `prod-*` → `muxly`.

## Wrapper examples

See per-service wrappers in each repo's `.github/workflows/`. Template:

```yaml
name: Publish
on:
  pull_request:
    paths: [cmd/**, internal/**, go.mod, go.sum, Dockerfile]
  push:
    branches: [main]
    paths: [cmd/**, internal/**, go.mod, go.sum, Dockerfile]

permissions:
  contents: read
  packages: write

jobs:
  publish:
    uses: Muxly-Corp/muxly-ci/.github/workflows/publish-go.yml@v1
    with:
      image_name: my-service
    secrets: inherit
```
