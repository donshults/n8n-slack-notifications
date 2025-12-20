# n8n Notification System Design

## Overview

This document describes a centralized notification system built with n8n that decouples project notifications from individual codebases. All projects send events to n8n via webhooks, and n8n handles routing, formatting, and delivery to Slack (and future channels).

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              YOUR PROJECTS                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  Context Vault    │    Future Project A    │    Future Project B            │
│  (GitHub Actions) │    (GitHub Actions)    │    (GitLab CI)                 │
└────────┬──────────┴──────────┬─────────────┴──────────┬─────────────────────┘
         │                     │                        │
         │ POST /webhook/notify│                        │
         ▼                     ▼                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           n8n NOTIFICATION HUB                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────┐                                                       │
│  │ Webhook Receiver │◄─── Single entry point for all notifications          │
│  │ (with auth)      │                                                       │
│  └────────┬─────────┘                                                       │
│           │                                                                  │
│           ▼                                                                  │
│  ┌──────────────────┐                                                       │
│  │ Event Classifier │◄─── Determine event type, severity, project           │
│  └────────┬─────────┘                                                       │
│           │                                                                  │
│           ▼                                                                  │
│  ┌──────────────────┐     ┌─────────────────────────────────────────┐      │
│  │ Config Lookup    │────►│ Project Configuration (Airtable/Sheet)  │      │
│  │                  │     │ - Channel mappings                      │      │
│  └────────┬─────────┘     │ - Notification preferences              │      │
│           │               │ - Team mentions                          │      │
│           │               └─────────────────────────────────────────┘      │
│           ▼                                                                  │
│  ┌──────────────────┐                                                       │
│  │ Routing Engine   │◄─── Switch node: route by project + event type        │
│  │ (Switch Node)    │                                                       │
│  └────────┬─────────┘                                                       │
│           │                                                                  │
│           ├──────────────┬──────────────┬──────────────┐                    │
│           ▼              ▼              ▼              ▼                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ Build       │  │ Deploy      │  │ Alert       │  │ Approval    │        │
│  │ Formatter   │  │ Formatter   │  │ Formatter   │  │ Formatter   │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│         │                │                │                │                │
│         └────────────────┴────────────────┴────────────────┘                │
│                          │                                                   │
│                          ▼                                                   │
│                 ┌──────────────────┐                                        │
│                 │ Slack Dispatcher │◄─── Send to appropriate channel        │
│                 └────────┬─────────┘                                        │
│                          │                                                   │
└──────────────────────────┼──────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SLACK                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  #deployments  │  #alerts  │  #context-vault  │  #general                   │
└─────────────────────────────────────────────────────────────────────────────┘
                           │
                           │ Button clicks, slash commands
                           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      n8n INTERACTION HANDLER                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐                                                       │
│  │ Slack Webhook    │◄─── Receives button clicks, slash commands            │
│  └────────┬─────────┘                                                       │
│           │                                                                  │
│           ▼                                                                  │
│  ┌──────────────────┐                                                       │
│  │ Action Router    │◄─── Route by action_id                                │
│  └────────┬─────────┘                                                       │
│           │                                                                  │
│           ├──────────────┬──────────────┬──────────────┐                    │
│           ▼              ▼              ▼              ▼                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ Approve     │  │ Reject      │  │ View Logs   │  │ Rollback    │        │
│  │ Deploy      │  │ Deploy      │  │             │  │             │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│         │                │                │                │                │
│         └────────────────┴────────────────┴────────────────┘                │
│                          │                                                   │
│                          ▼                                                   │
│                 ┌──────────────────┐                                        │
│                 │ Project Callback │◄─── Trigger actions in source project  │
│                 │ (webhook to GH)  │                                        │
│                 └──────────────────┘                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Standardized Event Payload

All projects send notifications using this standard format:

```json
{
  "event_type": "deploy",
  "status": "success",
  "project": {
    "id": "context-vault",
    "name": "Context Vault",
    "repo": "donshults/ThisIsDonsBrain"
  },
  "environment": "staging",
  "actor": {
    "name": "Don Shults",
    "github_username": "donshults"
  },
  "commit": {
    "sha": "abc123def",
    "message": "Add R2 storage integration",
    "url": "https://github.com/donshults/ThisIsDonsBrain/commit/abc123"
  },
  "pipeline": {
    "id": "12345",
    "url": "https://github.com/donshults/ThisIsDonsBrain/actions/runs/12345"
  },
  "timestamp": "2024-12-20T15:30:00Z",
  "metadata": {
    "duration_seconds": 180,
    "test_count": 42,
    "tests_passed": 42
  },
  "callback_url": "https://api.github.com/repos/donshults/ThisIsDonsBrain/dispatches"
}
```

### Event Types

| Event Type | Description | Status Values |
|------------|-------------|---------------|
| `test` | Test suite execution | `started`, `success`, `failed` |
| `build` | Build/compile step | `started`, `success`, `failed` |
| `deploy` | Deployment to environment | `started`, `success`, `failed`, `rolled_back` |
| `approval` | Deployment approval request | `pending`, `approved`, `rejected` |
| `alert` | Error or warning notification | `warning`, `error`, `critical` |
| `info` | Informational message | `info` |

---

## n8n Workflows

### Workflow 1: Notification Receiver (Main Entry Point)

**Trigger:** Webhook at `/webhook/notify`

**Purpose:** Receive all project notifications and route to appropriate handler

```
Webhook
  ↓
Validate Auth Token (Code Node)
  ↓
Log Event (for debugging/audit)
  ↓
Lookup Project Config (HTTP Request or Airtable)
  ↓
Switch: Route by event_type
  ├── test → Execute Sub-workflow: Test Notifications
  ├── build → Execute Sub-workflow: Build Notifications
  ├── deploy → Execute Sub-workflow: Deploy Notifications
  ├── approval → Execute Sub-workflow: Approval Requests
  ├── alert → Execute Sub-workflow: Alerts
  └── fallback → Log unknown event type
```

### Workflow 2: Deploy Notifications (Sub-workflow)

**Input:** Event payload + project config

**Purpose:** Format and send deployment notifications

```
Start
  ↓
Switch: Route by status
  ├── started → Format "Deployment Started" message
  ├── success → Format "Deployment Succeeded" message (green)
  ├── failed → Format "Deployment Failed" message (red) + mention team
  └── rolled_back → Format "Rollback Complete" message (yellow)
  ↓
Add Action Buttons (if applicable)
  - View Logs
  - View Deployment
  - Rollback (for success)
  ↓
Determine Slack Channel (from config)
  ↓
Send to Slack
  ↓
Return confirmation
```

### Workflow 3: Approval Requests (Sub-workflow)

**Input:** Event payload + project config

**Purpose:** Send interactive approval requests

```
Start
  ↓
Format Approval Request Message
  - Show what's being deployed
  - Show commit details
  - Show who triggered it
  ↓
Add Interactive Buttons
  - ✅ Approve (action_id: approve_deploy_{project}_{run_id})
  - ❌ Reject (action_id: reject_deploy_{project}_{run_id})
  - 📋 View Changes (url button)
  ↓
Determine Approvers (from config)
  ↓
Send to Approval Channel (mention approvers)
  ↓
Store Request State (for tracking)
```

### Workflow 4: Slack Interaction Handler

**Trigger:** Webhook for Slack interactive components

**Purpose:** Handle button clicks and slash commands

```
Webhook (Slack interaction)
  ↓
Parse Slack Payload
  ↓
Switch: Route by action_id prefix
  ├── approve_deploy_* → Process Approval
  │     ├── Verify user has permission
  │     ├── Call project callback URL
  │     ├── Update original message (remove buttons)
  │     └── Send confirmation
  ├── reject_deploy_* → Process Rejection
  │     ├── Update original message
  │     └── Send rejection notice
  ├── view_logs_* → Generate logs link
  └── rollback_* → Trigger rollback workflow
```

### Workflow 5: Error/Alert Handler

**Trigger:** Called from Workflow 1 or directly for critical alerts

**Purpose:** Handle errors with appropriate escalation

```
Start
  ↓
Determine Severity
  ↓
Switch: Route by severity
  ├── critical → Immediate Slack + @channel mention
  ├── error → Slack notification + responsible team
  └── warning → Slack notification (no mention)
  ↓
Format Error Details
  - Error message
  - Stack trace (if available)
  - Affected component
  - Link to logs
  ↓
Send to Alerts Channel
  ↓
Log for analysis (optional: send to error tracking service)
```

---

## Project Configuration Store

Store project-specific settings in Airtable, Google Sheets, or n8n's built-in storage:

### Configuration Schema

```json
{
  "project_id": "context-vault",
  "name": "Context Vault",
  "active": true,
  "channels": {
    "default": "#context-vault",
    "deploys": "#deployments",
    "alerts": "#alerts",
    "approvals": "#deployment-approvals"
  },
  "notifications": {
    "test_success": false,
    "test_failure": true,
    "deploy_started": true,
    "deploy_success": true,
    "deploy_failure": true
  },
  "mentions": {
    "on_failure": "@channel",
    "approvers": ["@don"]
  },
  "callback_auth": {
    "type": "bearer",
    "token_secret_name": "CONTEXT_VAULT_CALLBACK_TOKEN"
  }
}
```

### Configuration Options

| Setting | Type | Description |
|---------|------|-------------|
| `channels.default` | string | Default channel for notifications |
| `channels.deploys` | string | Channel for deployment notifications |
| `channels.alerts` | string | Channel for error alerts |
| `notifications.*` | boolean | Enable/disable specific notification types |
| `mentions.on_failure` | string | Who to mention when something fails |
| `mentions.approvers` | array | Who can approve production deploys |

---

## GitHub Actions Integration

### Update GitHub Actions to Use n8n

Replace the Slack GitHub Action with a simple webhook call:

```yaml
# In .github/workflows/deploy-staging.yml

- name: Notify n8n - Deploy Started
  run: |
    curl -X POST "${{ secrets.N8N_WEBHOOK_URL }}" \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer ${{ secrets.N8N_AUTH_TOKEN }}" \
      -d '{
        "event_type": "deploy",
        "status": "started",
        "project": {
          "id": "context-vault",
          "name": "Context Vault",
          "repo": "${{ github.repository }}"
        },
        "environment": "staging",
        "actor": {
          "name": "${{ github.actor }}",
          "github_username": "${{ github.actor }}"
        },
        "commit": {
          "sha": "${{ github.sha }}",
          "message": "${{ github.event.head_commit.message }}",
          "url": "${{ github.event.head_commit.url }}"
        },
        "pipeline": {
          "id": "${{ github.run_id }}",
          "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
        },
        "timestamp": "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'"
      }'

# ... deployment steps ...

- name: Notify n8n - Deploy Success
  if: success()
  run: |
    curl -X POST "${{ secrets.N8N_WEBHOOK_URL }}" \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer ${{ secrets.N8N_AUTH_TOKEN }}" \
      -d '{
        "event_type": "deploy",
        "status": "success",
        "project": { "id": "context-vault", "repo": "${{ github.repository }}" },
        "environment": "staging",
        "commit": { "sha": "${{ github.sha }}" },
        "pipeline": { "id": "${{ github.run_id }}" },
        "timestamp": "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'"
      }'

- name: Notify n8n - Deploy Failed
  if: failure()
  run: |
    curl -X POST "${{ secrets.N8N_WEBHOOK_URL }}" \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer ${{ secrets.N8N_AUTH_TOKEN }}" \
      -d '{
        "event_type": "deploy",
        "status": "failed",
        "project": { "id": "context-vault", "repo": "${{ github.repository }}" },
        "environment": "staging",
        "commit": { "sha": "${{ github.sha }}" },
        "pipeline": { "id": "${{ github.run_id }}" },
        "timestamp": "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'"
      }'
```

### Reusable Workflow for Notifications

Create a reusable workflow to simplify notification calls:

```yaml
# .github/workflows/notify.yml
name: Send Notification

on:
  workflow_call:
    inputs:
      event_type:
        required: true
        type: string
      status:
        required: true
        type: string
      environment:
        required: false
        type: string
        default: ''
    secrets:
      N8N_WEBHOOK_URL:
        required: true
      N8N_AUTH_TOKEN:
        required: true

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Send to n8n
        run: |
          curl -X POST "${{ secrets.N8N_WEBHOOK_URL }}" \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer ${{ secrets.N8N_AUTH_TOKEN }}" \
            -d '{
              "event_type": "${{ inputs.event_type }}",
              "status": "${{ inputs.status }}",
              "project": {
                "id": "context-vault",
                "repo": "${{ github.repository }}"
              },
              "environment": "${{ inputs.environment }}",
              "actor": { "github_username": "${{ github.actor }}" },
              "commit": {
                "sha": "${{ github.sha }}",
                "url": "${{ github.event.head_commit.url }}"
              },
              "pipeline": {
                "id": "${{ github.run_id }}",
                "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
              },
              "timestamp": "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'"
            }'
```

Usage in other workflows:

```yaml
jobs:
  deploy:
    # ... deployment steps ...

  notify-success:
    needs: deploy
    if: success()
    uses: ./.github/workflows/notify.yml
    with:
      event_type: deploy
      status: success
      environment: staging
    secrets: inherit

  notify-failure:
    needs: deploy
    if: failure()
    uses: ./.github/workflows/notify.yml
    with:
      event_type: deploy
      status: failed
      environment: staging
    secrets: inherit
```

---

## Slack Message Templates

### Deployment Success

```json
{
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "✅ Deployed to Staging"
      }
    },
    {
      "type": "section",
      "fields": [
        { "type": "mrkdwn", "text": "*Project:*\nContext Vault" },
        { "type": "mrkdwn", "text": "*Environment:*\nstaging" },
        { "type": "mrkdwn", "text": "*Commit:*\n<https://...|abc123d>" },
        { "type": "mrkdwn", "text": "*By:*\n@donshults" }
      ]
    },
    {
      "type": "context",
      "elements": [
        { "type": "mrkdwn", "text": "Add R2 storage integration" }
      ]
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "View Deployment" },
          "url": "https://github.com/.../actions/runs/123"
        },
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "Open Staging" },
          "url": "https://context-vault-staging.railway.app"
        }
      ]
    }
  ]
}
```

### Deployment Failed

```json
{
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "❌ Deployment Failed"
      }
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "<!channel> Staging deployment failed for *Context Vault*"
      }
    },
    {
      "type": "section",
      "fields": [
        { "type": "mrkdwn", "text": "*Commit:*\n<https://...|abc123d>" },
        { "type": "mrkdwn", "text": "*By:*\n@donshults" }
      ]
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "🔍 View Logs" },
          "url": "https://github.com/.../actions/runs/123",
          "style": "danger"
        }
      ]
    }
  ]
}
```

### Approval Request

```json
{
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "🚀 Production Deployment Approval"
      }
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Context Vault* is ready to deploy to production\n\n<@U123456> Please review and approve"
      }
    },
    {
      "type": "section",
      "fields": [
        { "type": "mrkdwn", "text": "*Commit:*\nabc123d" },
        { "type": "mrkdwn", "text": "*Requested by:*\n@donshults" },
        { "type": "mrkdwn", "text": "*Changes:*\n3 files, +150/-20" }
      ]
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "✅ Approve" },
          "style": "primary",
          "action_id": "approve_deploy_context-vault_12345"
        },
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "❌ Reject" },
          "style": "danger",
          "action_id": "reject_deploy_context-vault_12345"
        },
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "📋 View Changes" },
          "url": "https://github.com/.../compare/..."
        }
      ]
    }
  ]
}
```

---

## Implementation Phases

### Phase 1: Basic Notification Flow (Week 1)
1. Set up n8n instance (self-hosted or cloud)
2. Create main Webhook Receiver workflow
3. Create Deploy Notifications sub-workflow
4. Update GitHub Actions to call n8n webhook
5. Test with Context Vault staging deployments

**Deliverables:**
- Working n8n webhook endpoint
- Deployment notifications in Slack
- GitHub Actions updated

### Phase 2: Configuration & Routing (Week 2)
1. Set up project configuration store (Airtable or Sheet)
2. Implement config lookup in n8n
3. Add test/build notification workflows
4. Implement channel routing based on config
5. Add error handling workflow

**Deliverables:**
- Configurable channel routing
- Multiple notification types supported
- Error handling with logging

### Phase 3: Interactive Features (Week 3)
1. Set up Slack App for interactions
2. Create Slack Interaction Handler workflow
3. Implement approval request flow
4. Add button click handling
5. Set up callback to trigger GitHub Actions

**Deliverables:**
- Interactive approval buttons
- Production deployment gating via Slack
- Rollback button functionality

### Phase 4: Polish & Scale (Week 4)
1. Add slash commands (/deploy-status, /recent-deploys)
2. Implement notification digests for high-volume events
3. Add message threading for related notifications
4. Create dashboard/status workflow
5. Document system for team use

**Deliverables:**
- Slash command support
- Complete documentation
- Ready for additional projects

---

## Required GitHub Secrets

After implementing n8n, add these secrets:

| Secret | Description |
|--------|-------------|
| `N8N_WEBHOOK_URL` | URL of your n8n notification webhook |
| `N8N_AUTH_TOKEN` | Bearer token for authenticating with n8n |

Remove or keep as fallback:

| Secret | Status |
|--------|--------|
| `SLACK_WEBHOOK_URL` | Can be removed once n8n is working |

---

## Benefits of This Architecture

1. **Single Point of Control** - All notification logic in one place
2. **Easy Evolution** - Change routing rules without touching project code
3. **Multi-Project Support** - Same infrastructure for all projects
4. **Bi-directional** - Slack can trigger actions in projects
5. **Audit Trail** - All notifications logged in n8n
6. **Flexible Routing** - Route by project, severity, environment, time of day
7. **Rich Formatting** - Consistent, professional Slack messages
8. **Future-Proof** - Easy to add email, SMS, or other channels later

---

## Next Steps

1. **Decision:** Self-hosted n8n or n8n Cloud?
2. **Decision:** Configuration store - Airtable, Google Sheets, or n8n internal?
3. **Action:** Set up n8n instance
4. **Action:** Create Slack App for interactive components
5. **Action:** Import/create initial workflows
