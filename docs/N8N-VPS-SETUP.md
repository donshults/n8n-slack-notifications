# n8n Setup Guide - Using Existing Instance

Your n8n instance is already running at: **https://nca-n8n.callteksupport.net**

This guide covers setting up the CI/CD notification workflows on your existing n8n instance.

## Overview

We'll create a dedicated project/folder structure for notification workflows to keep them organized and separate from other automations.

---

## Step 1: Create Project Structure in n8n

1. Log in to your n8n instance: https://nca-n8n.callteksupport.net

2. **Create a new Project** (if using n8n 1.0+):
   - Click on the project dropdown (top left)
   - Click **+ Create Project**
   - Name: `CI-CD Notifications`
   - Description: `Centralized notification hub for all project deployments`

3. **Create Folders** within the project:
   - `Receivers` - Webhook entry points
   - `Formatters` - Message formatting sub-workflows
   - `Handlers` - Slack interaction handlers

   *Note: If your n8n version doesn't support projects/folders, use workflow naming conventions instead:*
   - `[CICD] Notification Receiver`
   - `[CICD] Deploy Notifications`
   - `[CICD] Slack Interactions`

---

## Step 2: Import Workflows

The workflow JSON files are in the `n8n-workflows/` directory of this repository.

### Import Order

Import workflows in this order (dependencies matter):

| Order | File | Purpose |
|-------|------|---------|
| 1 | `02-deploy-notifications.json` | Sub-workflow (no dependencies) |
| 2 | `03-slack-interactions.json` | Slack button handler |
| 3 | `01-notification-receiver.json` | Main entry point (calls others) |

### Import Steps

1. In n8n, go to **Workflows** → **Import from File**
2. Select the JSON file
3. Click **Import**
4. Move the workflow to the `CI-CD Notifications` project/folder
5. Repeat for each file

---

## Step 3: Configure Credentials

### Slack OAuth2 Credential

1. Go to **Credentials** → **Add Credential**
2. Search for **Slack OAuth2 API**
3. Enter your Bot Token (from Slack App - see [SLACK-APP-SETUP.md](SLACK-APP-SETUP.md))
4. Name it: `Slack - CI/CD Bot`
5. Save

### Webhook Authentication (Optional but Recommended)

Create a token for GitHub Actions to authenticate:

```bash
# Generate a secure token
openssl rand -hex 32
```

1. Go to **Credentials** → **Add Credential**
2. Search for **Header Auth**
3. Configure:
   - Name: `n8n Webhook Auth`
   - Header Name: `Authorization`
   - Header Value: `Bearer YOUR_GENERATED_TOKEN`
4. Save

---

## Step 4: Update Workflow Configuration

### Update Sub-workflow IDs

After importing, you need to link the workflows together:

1. Open `01-notification-receiver.json` (Notification Receiver)
2. Find the **Execute Workflow** nodes
3. Update the workflow IDs to match your imported workflows:
   - `Deploy Notifications` → Select the imported deploy workflow
   - (Add other sub-workflows as you create them)

### Update Slack Credentials

1. Open each workflow that has a Slack node
2. Click on the Slack node
3. Select your `Slack - CI/CD Bot` credential
4. Save

### Configure Channels

Edit the `02-deploy-notifications.json` workflow to set your Slack channels:

In the "Apply Config" code node, update:
```javascript
const defaultConfig = {
  channels: {
    default: '#your-deployments-channel',
    deploys: '#your-deployments-channel',
    alerts: '#your-alerts-channel',
    approvals: '#your-approvals-channel'
  },
  // ...
};
```

---

## Step 5: Activate Workflows

1. Open each workflow
2. Toggle the **Active** switch (top right)
3. Note the webhook URLs that are generated

### Your Webhook URLs

After activation, your webhooks will be at:

| Webhook | URL |
|---------|-----|
| Notification Receiver | `https://nca-n8n.callteksupport.net/webhook/notify` |
| Slack Interactions | `https://nca-n8n.callteksupport.net/webhook/slack-interactions` |

---

## Step 6: Test the Setup

### Test Notification Webhook

```bash
curl -X POST https://nca-n8n.callteksupport.net/webhook/notify \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "event_type": "deploy",
    "status": "success",
    "project": {
      "id": "context-vault",
      "name": "Context Vault"
    },
    "environment": "staging",
    "actor": {
      "github_username": "donshults"
    },
    "commit": {
      "sha": "abc123def456",
      "message": "Test notification"
    },
    "pipeline": {
      "id": "12345",
      "url": "https://github.com/donshults/ThisIsDonsBrain/actions/runs/12345"
    }
  }'
```

### Expected Response

```json
{
  "received": true,
  "event_type": "deploy",
  "project": "context-vault"
}
```

### Check Slack

You should see a message in your deployments channel.

---

## Folder Structure Summary

```
n8n Instance: nca-n8n.callteksupport.net
│
└── Project: CI-CD Notifications
    │
    ├── Folder: Receivers
    │   └── Notification Receiver (webhook: /notify)
    │
    ├── Folder: Formatters
    │   ├── Deploy Notifications (sub-workflow)
    │   ├── Test Notifications (future)
    │   └── Alert Notifications (future)
    │
    └── Folder: Handlers
        └── Slack Interactions (webhook: /slack-interactions)
```

---

## Adding New Projects

When you add a new project that needs notifications:

1. **No workflow changes needed** - The receiver handles all projects
2. **Add project config** to Airtable (when set up) or update the default config
3. **Add GitHub secrets** to the new repo:
   - `N8N_WEBHOOK_URL`: `https://nca-n8n.callteksupport.net/webhook/notify`
   - `N8N_AUTH_TOKEN`: Your webhook auth token

---

## Maintenance

### View Execution Logs

1. In n8n, click **Executions** (left sidebar)
2. Filter by workflow or date
3. Click on any execution to see details

### Update Workflows

1. Edit workflow in n8n
2. Test with manual trigger
3. Save changes
4. Workflows auto-update (no restart needed)

### Backup Workflows

Periodically export workflows as JSON:

1. Open workflow
2. Click **...** menu → **Download**
3. Save to `n8n-workflows/` in this repo
4. Commit changes

---

## Next Steps

1. ✅ n8n instance ready (existing)
2. → **Create Slack App** (see [SLACK-APP-SETUP.md](SLACK-APP-SETUP.md))
3. → **Import workflows** to n8n
4. → **Configure credentials**
5. → **Test webhook**
6. → **Update GitHub Actions** to use n8n
