# n8n Notification Integration Guide

This guide explains how to integrate any project with the centralized n8n notification system for CI/CD events.

## Overview

The n8n notification system provides:
- Centralized Slack notifications for all projects
- Consistent message formatting
- Easy extensibility for new event types
- No per-project Slack configuration needed

**Webhook URL:** `https://nca-n8n.callteksupport.net/webhook/notify`

---

## Quick Start (5 minutes)

### 1. Add GitHub Secret

In your GitHub repository, go to **Settings → Secrets and variables → Actions** and add:

| Secret Name | Value |
|-------------|-------|
| `N8N_WEBHOOK_URL` | `https://nca-n8n.callteksupport.net/webhook/notify` |

### 2. Add Notification Step to Workflow

Add this step to your GitHub Actions workflow:

```yaml
- name: Notify n8n
  run: |
    curl -s -X POST ${{ secrets.N8N_WEBHOOK_URL }} \
      -H "Content-Type: application/json" \
      -d '{
        "event_type": "deploy",
        "status": "success",
        "project": {
          "id": "your-project-id",
          "name": "Your Project Name"
        },
        "environment": "staging",
        "actor": {
          "github_username": "${{ github.actor }}"
        },
        "commit": {
          "sha": "${{ github.sha }}",
          "message": "${{ github.event.head_commit.message }}",
          "url": "${{ github.server_url }}/${{ github.repository }}/commit/${{ github.sha }}"
        },
        "pipeline": {
          "id": "${{ github.run_id }}",
          "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
        }
      }'
```

That's it! Your project will now send notifications through the centralized system.

---

## Event Payload Schema

### Required Fields

```json
{
  "event_type": "deploy | test | alert",
  "status": "started | success | failed",
  "project": {
    "id": "unique-project-slug",
    "name": "Human Readable Name"
  }
}
```

### Full Schema

```json
{
  "event_type": "deploy",
  "status": "success",
  "project": {
    "id": "context-vault",
    "name": "Context Vault"
  },
  "environment": "staging | production",
  "actor": {
    "github_username": "donshults",
    "name": "Don Shults"
  },
  "commit": {
    "sha": "abc123def456",
    "message": "Add new feature",
    "url": "https://github.com/org/repo/commit/abc123"
  },
  "pipeline": {
    "id": "12345",
    "url": "https://github.com/org/repo/actions/runs/12345"
  }
}
```

### Event Types

| Event Type | Description | Supported Statuses |
|------------|-------------|-------------------|
| `deploy` | Deployment events | `started`, `success`, `failed` |
| `test` | Test run events | `success`, `failed` |
| `alert` | System alerts | `warning`, `critical`, `resolved` |

---

## Complete Workflow Examples

### Example 1: Deploy to Staging on Push

```yaml
name: Deploy to Staging

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Notify n8n - Deploy Started
        run: |
          curl -s -X POST ${{ secrets.N8N_WEBHOOK_URL }} \
            -H "Content-Type: application/json" \
            -d '{
              "event_type": "deploy",
              "status": "started",
              "project": {
                "id": "my-project",
                "name": "My Project"
              },
              "environment": "staging",
              "actor": { "github_username": "${{ github.actor }}" },
              "commit": {
                "sha": "${{ github.sha }}",
                "message": "${{ github.event.head_commit.message }}",
                "url": "${{ github.server_url }}/${{ github.repository }}/commit/${{ github.sha }}"
              },
              "pipeline": {
                "id": "${{ github.run_id }}",
                "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
              }
            }'

      # Your deployment steps here
      - name: Deploy
        run: echo "Deploying..."

      - name: Notify n8n - Deploy Success
        run: |
          curl -s -X POST ${{ secrets.N8N_WEBHOOK_URL }} \
            -H "Content-Type: application/json" \
            -d '{
              "event_type": "deploy",
              "status": "success",
              "project": { "id": "my-project", "name": "My Project" },
              "environment": "staging",
              "actor": { "github_username": "${{ github.actor }}" },
              "commit": {
                "sha": "${{ github.sha }}",
                "message": "${{ github.event.head_commit.message }}",
                "url": "${{ github.server_url }}/${{ github.repository }}/commit/${{ github.sha }}"
              },
              "pipeline": {
                "id": "${{ github.run_id }}",
                "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
              }
            }'

      - name: Notify n8n - Deploy Failed
        if: failure()
        run: |
          curl -s -X POST ${{ secrets.N8N_WEBHOOK_URL }} \
            -H "Content-Type: application/json" \
            -d '{
              "event_type": "deploy",
              "status": "failed",
              "project": { "id": "my-project", "name": "My Project" },
              "environment": "staging",
              "actor": { "github_username": "${{ github.actor }}" },
              "commit": {
                "sha": "${{ github.sha }}",
                "message": "${{ github.event.head_commit.message }}",
                "url": "${{ github.server_url }}/${{ github.repository }}/commit/${{ github.sha }}"
              },
              "pipeline": {
                "id": "${{ github.run_id }}",
                "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
              }
            }'
```

### Example 2: Test Notifications (Failures Only)

```yaml
name: Test

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: npm test

      - name: Notify n8n - Test Failed
        if: failure()
        run: |
          curl -s -X POST ${{ secrets.N8N_WEBHOOK_URL }} \
            -H "Content-Type: application/json" \
            -d '{
              "event_type": "test",
              "status": "failed",
              "project": {
                "id": "my-project",
                "name": "My Project"
              },
              "actor": { "github_username": "${{ github.actor }}" },
              "commit": {
                "sha": "${{ github.sha }}",
                "message": "${{ github.event.head_commit.message }}",
                "url": "${{ github.server_url }}/${{ github.repository }}/commit/${{ github.sha }}"
              },
              "pipeline": {
                "id": "${{ github.run_id }}",
                "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
              }
            }'
```

### Example 3: Using a Reusable Notification Action

For cleaner workflows, create a composite action:

**.github/actions/notify-n8n/action.yml**
```yaml
name: 'Notify n8n'
description: 'Send notification to n8n webhook'
inputs:
  webhook_url:
    description: 'n8n webhook URL'
    required: true
  event_type:
    description: 'Event type (deploy, test, alert)'
    required: true
  status:
    description: 'Status (started, success, failed)'
    required: true
  project_id:
    description: 'Project ID'
    required: true
  project_name:
    description: 'Project name'
    required: true
  environment:
    description: 'Environment (staging, production)'
    required: false
    default: ''

runs:
  using: 'composite'
  steps:
    - name: Send notification
      shell: bash
      run: |
        curl -s -X POST ${{ inputs.webhook_url }} \
          -H "Content-Type: application/json" \
          -d '{
            "event_type": "${{ inputs.event_type }}",
            "status": "${{ inputs.status }}",
            "project": {
              "id": "${{ inputs.project_id }}",
              "name": "${{ inputs.project_name }}"
            },
            "environment": "${{ inputs.environment }}",
            "actor": { "github_username": "${{ github.actor }}" },
            "commit": {
              "sha": "${{ github.sha }}",
              "url": "${{ github.server_url }}/${{ github.repository }}/commit/${{ github.sha }}"
            },
            "pipeline": {
              "id": "${{ github.run_id }}",
              "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
          }'
```

**Usage in workflows:**
```yaml
- name: Notify n8n
  uses: ./.github/actions/notify-n8n
  with:
    webhook_url: ${{ secrets.N8N_WEBHOOK_URL }}
    event_type: deploy
    status: success
    project_id: my-project
    project_name: My Project
    environment: staging
```

---

## Testing Your Integration

### Test with curl

```bash
curl -X POST https://nca-n8n.callteksupport.net/webhook/notify \
  -H "Content-Type: application/json" \
  -d '{
    "event_type": "deploy",
    "status": "success",
    "project": {
      "id": "test-project",
      "name": "Test Project"
    },
    "environment": "staging",
    "actor": {
      "github_username": "your-username"
    },
    "commit": {
      "sha": "abc123",
      "message": "Test notification"
    }
  }'
```

### Expected Response

```json
{
  "received": true,
  "event_type": "deploy",
  "project": "test-project"
}
```

### Verify in Slack

Check the `#deployments` channel (or your configured channel) for the notification.

---

## Slack Message Format

The system formats messages based on event type and status:

### Deploy Started
```
🚀 Deploying to staging
Project: My Project
Commit: abc123
By: donshults
```

### Deploy Success
```
✅ Deployed to staging
Project: My Project
Commit: abc123
By: donshults
Message: Add new feature
View Pipeline
```

### Deploy Failed
```
❌ Deployment Failed @channel
Project: My Project
Environment: staging
Commit: abc123
By: donshults
View Logs
```

---

## Troubleshooting

### Notification not appearing in Slack

1. **Check webhook response** - Should return `{"received": true, ...}`
2. **Verify n8n workflow is active** - Check the Notification Receiver workflow in n8n
3. **Check Slack channel exists** - Ensure `#deployments` channel exists and bot is invited
4. **Review n8n execution logs** - In n8n, go to Executions to see any errors

### JSON parsing errors

- Ensure commit messages don't contain unescaped quotes
- Use proper JSON escaping for special characters
- Test payload with `jq` before sending: `echo '{"test": "value"}' | jq .`

### GitHub Actions secret not found

- Verify `N8N_WEBHOOK_URL` is set in repository secrets
- Check secret name matches exactly (case-sensitive)

---

## Architecture Reference

```
┌─────────────────┐     POST /webhook/notify     ┌─────────────────┐
│  GitHub Actions │ ─────────────────────────────▶│   n8n Instance  │
│  (any project)  │                               │   (centralized) │
└─────────────────┘                               └────────┬────────┘
                                                           │
                                                           │ Routes by event_type
                                                           │
                                                  ┌────────▼────────┐
                                                  │ Deploy/Test/    │
                                                  │ Alert Handlers  │
                                                  └────────┬────────┘
                                                           │
                                                           │ Formats & sends
                                                           │
                                                  ┌────────▼────────┐
                                                  │      Slack      │
                                                  │   #deployments  │
                                                  └─────────────────┘
```

---

## Related Documentation

- [N8N-VPS-SETUP.md](N8N-VPS-SETUP.md) - n8n instance setup
- [SLACK-APP-SETUP.md](SLACK-APP-SETUP.md) - Slack app configuration
- [N8N-NOTIFICATION-SYSTEM.md](../design/N8N-NOTIFICATION-SYSTEM.md) - System design document

---

## Support

For issues with the notification system:
1. Check n8n execution logs at `https://nca-n8n.callteksupport.net`
2. Review this integration guide
3. Contact Don @ CallTek Support
