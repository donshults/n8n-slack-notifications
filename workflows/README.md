# n8n Workflows

This directory contains the n8n workflow JSON files for the notification system.

## Workflows

| File | Name | Purpose |
|------|------|---------|
| `01-notification-receiver.json` | Notification Receiver | Main webhook entry point - receives all events |
| `02-deploy-notifications.json` | Deploy Notifications | Formats and sends deployment messages to Slack (includes retry logic) |
| `03-slack-interactions.json` | Slack Interaction Handler | Handles button clicks from Slack messages (includes retry logic) |
| `04-error-handler.json` | Error Handler | Email fallback when Slack fails after retries |

## Import Order

Import workflows in this order (dependencies matter):

1. **`04-error-handler.json`** - Error handler with no dependencies
2. **`02-deploy-notifications.json`** - Sub-workflow with no dependencies
3. **`03-slack-interactions.json`** - Webhook for Slack interactions
4. **`01-notification-receiver.json`** - Main entry point that calls the others

## After Import

1. In the "Notification Receiver" workflow, update the "Deploy Notifications" Execute Workflow node to reference the imported workflow ID
2. Configure Slack credentials on all Slack nodes
3. **Configure Error Workflow** on workflows 01, 02, and 03:
   - Open each workflow > Settings (gear icon) > Error Workflow
   - Select "04 - Error Handler"
   - Save the workflow
4. **Configure Error Handler credentials** (04-error-handler only):
   - Connect AWS SES credential: "Amazon SES - Notifications"
   - Connect Airtable credential: "Airtable - n8n Notifications"
   - Update Airtable nodes with your Base ID and Table IDs
5. Activate the workflows that need webhooks:
   - ✅ Activate: Notification Receiver
   - ❌ Don't activate: Deploy Notifications (it's a sub-workflow)
   - ✅ Activate: Slack Interaction Handler
   - ❌ Don't activate: Error Handler (triggered by errors)

## Webhook URLs

After activation:

| Workflow | Webhook Path | Full URL |
|----------|--------------|----------|
| Notification Receiver | `/webhook/notify` | `https://your-n8n.com/webhook/notify` |
| Slack Interaction Handler | `/webhook/slack-interactions` | `https://your-n8n.com/webhook/slack-interactions` |

## Retry Logic & Error Handling

The deploy notifications (02) and slack interactions (03) workflows include built-in retry logic:

- **Retry Count**: 2 retries (3 total attempts)
- **Retry Interval**: 2 minutes between attempts
- **On Final Failure**: Triggers the error handler workflow

The error handler (04) provides email fallback:
- Sends notification email to project recipients (from Airtable)
- Sends admin alert to `don@callteksupport.com`
- Deduplicates alerts within 5-minute windows

See `docs/EMAIL-FALLBACK-SETUP.md` for complete configuration instructions.

## Compatibility

These workflows use n8n node versions compatible with n8n 1.x:
- Webhook v1
- IF v1
- Code v1
- Slack v1
- HTTP Request v1
- Wait v1
- Error Trigger
- AWS SES
- Airtable

If you're using a newer n8n version, the workflows will still work but you may see upgrade prompts.
