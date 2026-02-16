# Repository Dispatch Trigger Action

[![GitHub release](https://img.shields.io/github/release/sushidev-team/workflow-trigger-action.svg)](https://github.com/sushidev-team/workflow-trigger-action/releases)
[![GitHub marketplace](https://img.shields.io/badge/marketplace-workflow--trigger--action-blue?logo=github)](https://github.com/marketplace/actions/workflow-trigger-action)
[![Test Status](https://github.com/sushidev-team/workflow-trigger-action/workflows/Test%20Action/badge.svg)](https://github.com/sushidev-team/workflow-trigger-action/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A GitHub Action that triggers workflows via [`repository_dispatch`](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#repository_dispatch) with a single API call. No workflow enumeration, no rate limit stacking — just one dispatch event that all listening workflows pick up.

## Quick Start

### Caller workflow

```yaml
name: Deploy Staging
on: [push]

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - uses: sushidev-team/workflow-trigger-action@v2
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          event_type: 'docker-stage'
          client_payload: |
            {
              "version": "${{ github.sha }}",
              "environment": "staging"
            }
```

### Target workflow

Target workflows listen for the dispatched event type:

```yaml
name: Docker Stage Build
on:
  repository_dispatch:
    types: [docker-stage]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Version: ${{ github.event.client_payload.version }}"
          echo "Ref: ${{ github.event.client_payload.ref }}"
```

The `ref` input is automatically included in the `client_payload`.

## Inputs

| Parameter | Description | Required | Default |
|-----------|-------------|----------|---------|
| `github_token` | GitHub token for API access | Yes | — |
| `event_type` | The `repository_dispatch` event type target workflows listen for | Yes | — |
| `target_repo` | Target repository in `owner/repo` format | No | Current repository |
| `ref` | Git reference, passed inside `client_payload` | No | `master` |
| `client_payload` | JSON object sent as `client_payload` | No | `{}` |

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `triggered` | Whether the dispatch succeeded | `true` / `false` |

## Examples

### Cross-repository dispatch

```yaml
- uses: sushidev-team/workflow-trigger-action@v2
  with:
    github_token: ${{ secrets.DEPLOY_PAT }}
    target_repo: 'company/production-repo'
    event_type: 'deploy-prod'
    ref: ${{ github.ref_name }}
    client_payload: |
      {
        "version": "${{ github.sha }}",
        "triggered_by": "${{ github.actor }}"
      }
```

### Multiple event types via matrix

```yaml
jobs:
  dispatch:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        event: [build-images, run-tests, deploy-staging]
    steps:
      - uses: sushidev-team/workflow-trigger-action@v2
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          event_type: ${{ matrix.event }}
          client_payload: |
            {
              "version": "${{ github.sha }}"
            }
```

### Accessing payload in target workflows

All fields from `client_payload` are available under `github.event.client_payload`:

```yaml
on:
  repository_dispatch:
    types: [deploy-prod]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Version: ${{ github.event.client_payload.version }}"
          echo "Ref: ${{ github.event.client_payload.ref }}"
          echo "Triggered by: ${{ github.event.client_payload.triggered_by }}"
```

## Error Handling

- **Retry logic**: 3 attempts with exponential backoff (5s, 10s delays)
- **Validation**: The `client_payload` input is validated as JSON before dispatch
- The action exits with failure if all retry attempts are exhausted

## Token Permissions

The token needs the `repo` scope (for private repos) or `public_repo` scope (for public repos) to send `repository_dispatch` events. The default `GITHUB_TOKEN` works for dispatching to the same repository.

For cross-repository dispatch, use a Personal Access Token (PAT) or a GitHub App token with appropriate permissions.

## Migrating from v1

v2 is a breaking change that replaces `workflow_dispatch` (per-workflow API calls) with `repository_dispatch` (single API call). v1 remains available at `@v1` — no action needed if you don't want to upgrade.

### What changed

| v1 | v2 |
|----|-----|
| `workflow_prefix` input | Removed — no workflow enumeration |
| `cache_max_age` input | Removed — no caching needed |
| `workflow_inputs` input | Renamed to `client_payload` |
| — | `event_type` input (required) |
| `triggered_workflows` output | Removed |
| `workflow_ids` output | Removed |
| `triggered` output | New — `true`/`false` |

### Step 1: Update the caller workflow

```diff
 - uses: sushidev-team/workflow-trigger-action@v2
   with:
     github_token: ${{ secrets.GITHUB_TOKEN }}
-    workflow_prefix: 'docker-stage'
-    cache_max_age: '1800'
-    workflow_inputs: |
+    event_type: 'docker-stage'
+    client_payload: |
       {
         "version": "${{ github.sha }}"
       }
```

### Step 2: Update target workflows

Target workflows must switch their trigger from `workflow_dispatch` to `repository_dispatch`:

```diff
 on:
-  workflow_dispatch:
-    inputs:
-      version:
-        description: 'Version to deploy'
-        required: false
+  repository_dispatch:
+    types: [docker-stage]
```

And update how they read inputs:

```diff
-- echo "Version: ${{ github.event.inputs.version }}"
+- echo "Version: ${{ github.event.client_payload.version }}"
```

### Step 3: Update output references

```diff
-- echo "Triggered ${{ steps.deploy.outputs.triggered_workflows }} workflows"
-- echo "IDs: ${{ steps.deploy.outputs.workflow_ids }}"
+- echo "Triggered: ${{ steps.deploy.outputs.triggered }}"
```

## License

MIT License — see [LICENSE](LICENSE) for details.

Created and maintained by [Sushi Dev GmbH](https://github.com/sushidev-team).
