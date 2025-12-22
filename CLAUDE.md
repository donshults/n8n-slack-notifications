# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a centralized CI/CD notification system built with n8n that sends events from GitHub Actions to Slack. It provides a single webhook endpoint (`/webhook/notify`) that all projects can use, eliminating per-project Slack configuration.

**n8n Instance:** `https://nca-n8n.callteksupport.net`

## Architecture

```
GitHub Actions (any project)
       │
       │ POST /webhook/notify
       ▼
┌─────────────────────────────────────┐
│  n8n Notification Receiver          │
│  - Validates payload                │
│  - Routes by event_type             │
└─────────────┬───────────────────────┘
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
 Deploy    Test      Alert
 Handler   Handler   Handler
    │
    ▼
  Slack
```

**Workflow Flow:**
1. `01-notification-receiver.json` - Main webhook entry point, routes events
2. `02-deploy-notifications.json` - Sub-workflow for formatting deploy messages
3. `03-slack-interactions.json` - Handles Slack button clicks (approve/reject/rollback)

## Workflow Import Order

Workflows must be imported in this order due to dependencies:
1. `02-deploy-notifications.json` (sub-workflow, no dependencies)
2. `03-slack-interactions.json` (webhook for Slack)
3. `01-notification-receiver.json` (main entry point, calls the others)

After import, update the "Deploy Notifications" Execute Workflow node in the receiver to reference the correct workflow ID.

## Event Payload Schema

Required fields for webhook:
```json
{
  "event_type": "deploy | test | alert",
  "status": "started | success | failed",
  "project": {
    "id": "project-slug",
    "name": "Project Name"
  }
}
```

Full schema with optional fields:
- `environment`: staging | production
- `actor.github_username`: GitHub username
- `commit.sha`, `commit.message`, `commit.url`: Commit details
- `pipeline.id`, `pipeline.url`: GitHub Actions run info

## Key Files

| File | Purpose |
|------|---------|
| `workflows/*.json` | n8n workflow definitions (import to n8n, don't edit directly) |
| `docs/N8N-VPS-SETUP.md` | n8n instance configuration |
| `docs/SLACK-APP-SETUP.md` | Slack app configuration |
| `docs/PROJECT-INTEGRATION.md` | How to add this to any project |

## Configuration

Default channel routing is defined in the "Validate & Configure" Code node in `01-notification-receiver.json`:
- `#deployments` - Deploy notifications
- `#alerts` - Alert notifications
- `#deployment-approvals` - Approval requests

To customize: edit the `defaultConfig` object in the Code node or implement Airtable/Sheet lookup.

## Testing Webhooks

```bash
curl -X POST https://nca-n8n.callteksupport.net/webhook/notify \
  -H "Content-Type: application/json" \
  -d '{
    "event_type": "deploy",
    "status": "success",
    "project": {"id": "test", "name": "Test Project"},
    "environment": "staging"
  }'
```

Expected response: `{"received": true, "event_type": "deploy", "project": "test"}`

## Adding Projects

1. Add `N8N_WEBHOOK_URL` secret to the GitHub repo
2. Add curl webhook calls to GitHub Actions (see docs/PROJECT-INTEGRATION.md)
3. No workflow changes needed - receiver handles all projects
