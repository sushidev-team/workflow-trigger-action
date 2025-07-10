# 🚀 Workflow Trigger Action

[![GitHub release](https://img.shields.io/github/release/sushidev-team/workflow-trigger-action.svg)](https://github.com/sushidev-team/workflow-trigger-action/releases)
[![GitHub marketplace](https://img.shields.io/badge/marketplace-workflow--trigger--action-blue?logo=github)](https://github.com/marketplace/actions/workflow-trigger-action)
[![Test Status](https://github.com/sushidev-team/workflow-trigger-action/workflows/Test%20Action/badge.svg)](https://github.com/sushidev-team/workflow-trigger-action/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A robust and intelligent GitHub Action for triggering workflows with advanced caching mechanisms, fallback strategies, and retry logic.

## 🌟 Key Features

- **🧠 Smart Caching**: Combines file-based and ETag-based caching for optimal performance
- **🔄 Automatic Fallbacks**: Works even during API outages through local workflow detection  
- **⚡ Retry Mechanism**: Exponential backoff for rate-limiting and temporary failures
- **🎯 Flexible Filtering**: Precise workflow selection via name prefix matching
- **📊 Detailed Outputs**: Comprehensive information about triggered workflows
- **🛡️ Production-Ready**: Robust error handling for enterprise environments

## 📋 Table of Contents

- [Quick Start](#-quick-start)
- [Inputs & Outputs](#-inputs--outputs)
- [Usage Examples](#-usage-examples)
- [Caching System](#-caching-system)
- [Error Handling](#-error-handling)
- [Best Practices](#-best-practices)
- [Troubleshooting](#-troubleshooting)
- [Contributing](#-contributing)

## 🎯 Core Capabilities

### Intelligent Workflow Management
- **Automatic Discovery**: Finds workflows via GitHub API or local files
- **Prefix Filtering**: Triggers only workflows matching specific name patterns
- **Batch Processing**: Efficient handling of multiple workflows

### Performance Optimization
- **Multi-Level Caching**: 
  - File cache (30min default)
  - ETag-based validation
  - Automatic cache invalidation
- **Rate Limit Management**: Smart handling of API limits
- **Optimized Sequencing**: Efficient workflow trigger patterns

### Enterprise Features
- **Graceful Degradation**: Continues working during API issues
- **Comprehensive Logging**: Detailed status information
- **Monitoring Integration**: Outputs for external monitoring systems

## 🚀 Quick Start

### Basic Usage

```yaml
name: Deploy Staging
on: [push]

jobs:
  trigger-staging:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Trigger Docker Stage Workflows
        uses: sushidev-team/workflow-trigger-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          workflow_prefix: 'docker-stage'
```

### Advanced Configuration

```yaml
- name: Trigger Production Deployment
  id: deploy
  uses: sushidev-team/workflow-trigger-action@v1
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    target_repo: 'company/production-repo'
    workflow_prefix: 'deploy-prod'
    ref: ${{ github.ref_name }}
    cache_max_age: '3600'
    workflow_inputs: |
      {
        "version": "${{ github.sha }}",
        "environment": "production",
        "triggered_by": "${{ github.actor }}",
        "deployment_type": "automated"
      }

- name: Report Results
  run: |
    echo "✅ Triggered ${{ steps.deploy.outputs.triggered_workflows }} workflows"
    echo "📋 Workflow IDs: ${{ steps.deploy.outputs.workflow_ids }}"
```

## 📋 Inputs & Outputs

### Inputs

| Parameter | Description | Required | Default | Example |
|-----------|-------------|----------|---------|---------|
| `github_token` | GitHub token for API access | ✅ | - | `${{ secrets.GITHUB_TOKEN }}` |
| `target_repo` | Target repository (owner/repo format) | ❌ | `${{ github.repository }}` | `company/backend-services` |
| `workflow_prefix` | Workflow name prefix for filtering | ❌ | `docker-stage` | `deploy-`, `build-prod` |
| `ref` | Git reference to trigger workflows on | ❌ | `master` | `main`, `develop`, `v1.2.3` |
| `cache_max_age` | Cache validity in seconds | ❌ | `1800` | `3600` (1h), `7200` (2h) |
| `workflow_inputs` | JSON inputs for triggered workflows | ❌ | `{}` | See examples below |

### Outputs

| Output | Description | Type | Example |
|--------|-------------|------|---------|
| `triggered_workflows` | Number of workflows triggered | Number | `3` |
| `workflow_ids` | Comma-separated list of triggered workflow IDs | String | `12345,67890,11111` |

## 🔧 Usage Examples

### Cross-Repository Triggering

```yaml
name: Multi-Repo Deployment

jobs:
  deploy-microservices:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [user-service, payment-service, notification-service]
    steps:
      - name: Deploy ${{ matrix.service }}
        uses: sushidev-team/workflow-trigger-action@v1
        with:
          github_token: ${{ secrets.DEPLOY_TOKEN }}
          target_repo: company/${{ matrix.service }}
          workflow_prefix: 'deploy-production'
          workflow_inputs: |
            {
              "version": "${{ github.sha }}",
              "service_name": "${{ matrix.service }}",
              "environment": "production"
            }
```

### Conditional Triggering

```yaml
- name: Trigger Workflows Based on Changes
  uses: sushidev-team/workflow-trigger-action@v1
  if: contains(github.event.head_commit.message, '[deploy]')
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    workflow_prefix: ${{ github.event.head_commit.message contains '[prod]' && 'deploy-prod' || 'deploy-stage' }}
    workflow_inputs: |
      {
        "commit_message": "${{ github.event.head_commit.message }}",
        "author": "${{ github.event.head_commit.author.name }}"
      }
```

### Multiple Workflow Types

```yaml
jobs:
  trigger-pipeline:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        stage: 
          - { prefix: 'build-', environment: 'build' }
          - { prefix: 'test-', environment: 'testing' }
          - { prefix: 'deploy-stage-', environment: 'staging' }
          - { prefix: 'deploy-prod-', environment: 'production' }
    steps:
      - name: Trigger ${{ matrix.stage.environment }} workflows
        uses: sushidev-team/workflow-trigger-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          workflow_prefix: ${{ matrix.stage.prefix }}
          workflow_inputs: |
            {
              "stage": "${{ matrix.stage.environment }}",
              "version": "${{ github.sha }}"
            }
```

### Custom Workflow Inputs

```yaml
- name: Trigger with Dynamic Inputs
  uses: sushidev-team/workflow-trigger-action@v1
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    workflow_prefix: 'custom-deploy'
    workflow_inputs: |
      {
        "version": "${{ github.sha }}",
        "environment": "${{ github.ref_name == 'main' && 'production' || 'staging' }}",
        "feature_flags": {
          "new_ui": true,
          "beta_features": ${{ github.ref_name == 'develop' }}
        },
        "deployment_config": {
          "replicas": ${{ github.ref_name == 'main' && 5 || 2 }},
          "resources": {
            "cpu": "500m",
            "memory": "1Gi"
          }
        }
      }
```

## 🗂️ Caching System

The action implements a sophisticated two-tier caching system:

### File-Based Caching
- Stores workflow data locally for configurable duration
- Default: 30 minutes (`cache_max_age: 1800`)
- Survives across job runs within the same workflow

### ETag-Based Caching
- Uses GitHub API ETags to avoid unnecessary requests
- Automatically validates cache freshness
- Reduces API rate limit consumption

### Cache Behavior
```
Request Flow:
1. Check file cache age → Use if fresh
2. Check ETag cache → Send conditional request
3. API responds:
   - 304 Not Modified → Use cached data
   - 200 OK → Update cache with new data
   - 403/429/5xx → Fall back to local workflows
```

## 🛡️ Error Handling

### Retry Logic
- **Exponential Backoff**: 10s, 20s, 30s delays for rate limits
- **Rate Limit Handling**: Automatic retry with increasing delays
- **Network Failures**: Up to 3 retry attempts per workflow

### Fallback Mechanisms
- **API Failures**: Automatically switches to local workflow detection
- **Permission Issues**: Graceful degradation with informative logging
- **Missing Workflows**: Continues execution without failing the job

### Error Scenarios
```yaml
# The action handles these gracefully:
- GitHub API rate limits (403/429)
- Network timeouts
- Invalid repository access
- Missing workflow files
- Malformed workflow inputs
```

## 📚 Best Practices

### Security
```yaml
# Use dedicated tokens for cross-repo access
- name: Deploy to Production
  uses: sushidev-team/workflow-trigger-action@v1
  with:
    github_token: ${{ secrets.DEPLOY_PAT }}  # Personal Access Token
    target_repo: 'company/production-repo'
```

### Performance
```yaml
# Optimize caching for frequent workflows
- uses: sushidev-team/workflow-trigger-action@v1
  with:
    cache_max_age: '7200'  # 2 hours for stable workflows
    workflow_prefix: 'deploy-prod'
```

### Monitoring
```yaml
# Track deployment metrics
- name: Monitor Deployments
  if: always()
  run: |
    echo "::notice::Triggered ${{ steps.deploy.outputs.triggered_workflows }} workflows"
    echo "deployment_count=${{ steps.deploy.outputs.triggered_workflows }}" >> $GITHUB_ENV
```

## 🔧 Troubleshooting

### Common Issues

**Issue**: No workflows found with prefix
```bash
# Check if workflows exist
curl -H "Authorization: token $TOKEN" \
  https://api.github.com/repos/owner/repo/actions/workflows

# Verify workflow names in .github/workflows/
ls -la .github/workflows/
```

**Issue**: Permission denied (403)
```yaml
# Ensure token has sufficient permissions
permissions:
  actions: write
  contents: read
```

**Issue**: Rate limit exceeded
```yaml
# Increase cache duration and add delays
- uses: sushidev-team/workflow-trigger-action@v1
  with:
    cache_max_age: '3600'  # 1 hour cache
```

### Debug Mode
```yaml
# Enable verbose logging
- uses: sushidev-team/workflow-trigger-action@v1
  env:
    ACTIONS_STEP_DEBUG: true
```

## 🤝 Contributing

We welcome contributions! Please see our [Contributing Guide](CONTRIBUTING.md) for details.

### Development Setup
```bash
git clone https://github.com/sushidev-team/workflow-trigger-action.git
cd workflow-trigger-action

# Test locally
act -j test

# Run tests
npm test
```

### Reporting Issues
- Use the [issue tracker](https://github.com/sushidev-team/workflow-trigger-action/issues)
- Include workflow logs and error messages
- Specify your GitHub Actions runner environment

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🏆 Credits

Created and maintained by [Sushi Dev GmbH](https://github.com/sushidev-team).

### Contributors
- Add your name here by contributing!

## 📊 Changelog

### [v1.0.0] - 2025-01-XX
- Initial release
- Smart caching implementation
- Fallback mechanism for API failures
- Retry logic with exponential backoff
- Comprehensive error handling

---

**Star ⭐ this repository if you find it useful!**