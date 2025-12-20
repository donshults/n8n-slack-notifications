# n8n Workflows

This directory contains the n8n workflow JSON files for the notification system.

## Workflows

| File | Name | Purpose |
|------|------|---------|
| `01-notification-receiver.json` | Notification Receiver | Main webhook entry point - receives all events |
| `02-deploy-notifications.json` | Deploy Notifications | Formats and sends deployment messages to Slack |
| `03-slack-interactions.json` | Slack Interaction Handler | Handles button clicks from Slack messages |

## Import Order

Import workflows in this order (dependencies matter):

1. **`02-deploy-notifications.json`** - Sub-workflow with no dependencies
2. **`03-slack-interactions.json`** - Webhook for Slack interactions
3. **`01-notification-receiver.json`** - Main entry point that calls the others

## After Import

1. In the "Notification Receiver" workflow, update the "Deploy Notifications" Execute Workflow node to reference the imported workflow ID
2. Configure Slack credentials on all Slack nodes
3. Activate the workflows that need webhooks:
   - ✅ Activate: Notification Receiver
   - ❌ Don't activate: Deploy Notifications (it's a sub-workflow)
   - ✅ Activate: Slack Interaction Handler

## Webhook URLs

After activation:

| Workflow | Webhook Path | Full URL |
|----------|--------------|----------|
| Notification Receiver | `/webhook/notify` | `https://your-n8n.com/webhook/notify` |
| Slack Interaction Handler | `/webhook/slack-interactions` | `https://your-n8n.com/webhook/slack-interactions` |

## Compatibility

These workflows use n8n node versions compatible with n8n 1.x:
- Webhook v1
- IF v1
- Code v1
- Slack v1
- HTTP Request v1

If you're using a newer n8n version, the workflows will still work but you may see upgrade prompts.
