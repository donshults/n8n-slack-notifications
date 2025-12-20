# n8n Slack Notifications

A centralized notification system built with n8n that sends CI/CD events to Slack. Designed to work across multiple projects with a single webhook endpoint.

## Features

- **Centralized notifications** - One n8n instance handles notifications for all your projects
- **Consistent formatting** - Professional Slack messages with project info, commit details, and action buttons
- **Easy integration** - Just add a webhook URL to your GitHub Actions
- **Extensible** - Add new event types, channels, or notification channels without touching project code
- **Interactive** - Support for Slack button clicks (approve, reject, rollback)

## Quick Start

### 1. Set up n8n workflows

Import the workflow JSON files from the `workflows/` directory into your n8n instance:

1. `02-deploy-notifications.json` (import first - it's a sub-workflow)
2. `03-slack-interactions.json`
3. `01-notification-receiver.json` (import last - it references the others)

See [docs/N8N-VPS-SETUP.md](docs/N8N-VPS-SETUP.md) for detailed setup instructions.

### 2. Create Slack App

Create a Slack App with bot permissions to send messages. See [docs/SLACK-APP-SETUP.md](docs/SLACK-APP-SETUP.md).

### 3. Add to your project

Add the `N8N_WEBHOOK_URL` secret to your GitHub repository:

```
Settings → Secrets and variables → Actions → New repository secret
Name: N8N_WEBHOOK_URL
Value: https://your-n8n-instance.com/webhook/notify
```

Then add notification steps to your workflow:

```yaml
- name: Notify n8n - Deploy Success
  run: |
    curl -s -X POST ${{ secrets.N8N_WEBHOOK_URL }} \
      -H "Content-Type: application/json" \
      -d '{
        "event_type": "deploy",
        "status": "success",
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
```

See [docs/PROJECT-INTEGRATION.md](docs/PROJECT-INTEGRATION.md) for complete integration guide.

## Repository Structure

```
n8n-slack-notifications/
├── README.md                 # This file
├── workflows/                # n8n workflow JSON files
│   ├── 01-notification-receiver.json
│   ├── 02-deploy-notifications.json
│   └── 03-slack-interactions.json
├── docs/
│   ├── N8N-VPS-SETUP.md      # n8n instance setup
│   ├── SLACK-APP-SETUP.md    # Slack app configuration
│   ├── PROJECT-INTEGRATION.md # How to integrate projects
│   └── SYSTEM-DESIGN.md      # Architecture overview
└── examples/
    └── github-actions/
        ├── deploy-staging.yml
        ├── deploy-production.yml
        ├── test.yml
        └── notify-n8n-action/  # Reusable composite action
```

## Event Types

| Event Type | Description | Statuses |
|------------|-------------|----------|
| `deploy` | Deployment events | `started`, `success`, `failed` |
| `test` | Test execution | `success`, `failed` |
| `alert` | System alerts | `warning`, `critical`, `resolved` |

## Payload Schema

```json
{
  "event_type": "deploy",
  "status": "success",
  "project": {
    "id": "project-slug",
    "name": "Project Display Name"
  },
  "environment": "staging",
  "actor": {
    "github_username": "username"
  },
  "commit": {
    "sha": "abc123",
    "message": "Commit message",
    "url": "https://github.com/..."
  },
  "pipeline": {
    "id": "12345",
    "url": "https://github.com/.../actions/runs/12345"
  }
}
```

## Slack Message Examples

### Deploy Started
```
🚀 Deploying to staging
Project: My Project
Commit: abc123
By: username
```

### Deploy Success
```
✅ Deployed to staging
Project: My Project
Commit: abc123
By: username
Message: Add new feature
[View Pipeline]
```

### Deploy Failed
```
❌ Deployment Failed @channel
Project: My Project
Environment: staging
Commit: abc123
By: username
[View Logs]
```

## Documentation

- [Project Integration Guide](docs/PROJECT-INTEGRATION.md) - How to add notifications to your project
- [n8n Setup Guide](docs/N8N-VPS-SETUP.md) - Setting up the n8n instance
- [Slack App Setup](docs/SLACK-APP-SETUP.md) - Creating and configuring the Slack app
- [System Design](docs/SYSTEM-DESIGN.md) - Architecture and design decisions

## Requirements

- n8n instance (self-hosted or cloud)
- Slack workspace with ability to create apps
- GitHub Actions (or any CI/CD that can make HTTP requests)

## License

MIT

## Author

Don @ CallTek Support
