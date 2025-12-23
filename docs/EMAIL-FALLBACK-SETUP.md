# Email Fallback Setup Guide

This guide covers the complete setup for the email fallback feature, which sends email notifications when Slack delivery fails after retry attempts.

## Overview

The email fallback system provides:
- **Retry Logic**: 2 automatic retries with 2-minute intervals before failing
- **Email Fallback**: Project-specific notification emails via Amazon SES
- **Admin Alerts**: Immediate notification to admin on any Slack failure
- **Deduplication**: 5-minute window prevents duplicate alert emails

## Prerequisites

Before starting, ensure you have:
- [ ] Access to the n8n instance at `https://nca-n8n.callteksupport.net`
- [ ] AWS account with SES access
- [ ] Airtable account
- [ ] Verified sender email in SES (e.g., `noreply@callteksupport.com`)

## Setup Steps

### Step 1: Airtable Configuration

Follow the detailed instructions in [AIRTABLE-SETUP.md](./AIRTABLE-SETUP.md) to:

1. Create the "n8n Notifications" base
2. Create the "Projects" table with columns:
   - `project_id` (Primary, Single line text)
   - `project_name` (Single line text)
   - `recipients` (Long text - comma-separated emails)
   - `notes` (Long text)
3. Create the "Error Deduplication" table with columns:
   - `dedup_key` (Formula: `{error_type} & "_" & {project_id}`)
   - `error_type` (Single line text)
   - `project_id` (Single line text)
   - `last_sent` (Date with time)
   - `count` (Number)
4. Add initial project entries
5. Generate API token with read/write access

**Record the Base ID** - you'll need it when configuring the workflows.

### Step 2: n8n Credential Configuration

Follow the detailed instructions in [N8N-CREDENTIALS-SETUP.md](./N8N-CREDENTIALS-SETUP.md) to:

1. Create AWS SES credential named **"Amazon SES - Notifications"**
   - Requires AWS Access Key ID and Secret Access Key
   - User must have `ses:SendEmail` and `ses:SendRawEmail` permissions
   - Sender email must be verified in SES

2. Create Airtable credential named **"Airtable - n8n Notifications"**
   - Requires Personal Access Token from Airtable
   - Token needs read/write access to the "n8n Notifications" base

### Step 3: Import Workflows

**Important: Backup existing workflows before importing!**

In n8n, go to **Workflows** and export the existing workflows as backup.

#### Import Order:

1. **`04-error-handler.json`** (new workflow)
   - Import: Settings (gear icon) > Import from File
   - Note the assigned workflow ID after import
   - Configure Airtable nodes with your Base ID and Table IDs
   - Connect credentials:
     - AWS SES nodes → "Amazon SES - Notifications"
     - Airtable nodes → "Airtable - n8n Notifications"

2. **`02-deploy-notifications.json`** (updated with retry logic)
   - Delete the existing workflow (after backup)
   - Import the updated version
   - Reconnect Slack credentials

3. **`03-slack-interactions.json`** (updated with retry logic)
   - Delete the existing workflow (after backup)
   - Import the updated version
   - Reconnect Slack credentials

### Step 4: Configure Error Workflow

For each of the following workflows, configure the error handler:

#### 01-notification-receiver.json:
1. Open the workflow in n8n
2. Click the gear icon (Settings) in the top right
3. Select "Error Workflow"
4. Choose "04 - Error Handler" from the dropdown
5. Save the workflow

#### 02-deploy-notifications.json:
1. Open the workflow
2. Settings > Error Workflow > Select "04 - Error Handler"
3. Save

#### 03-slack-interactions.json:
1. Open the workflow
2. Settings > Error Workflow > Select "04 - Error Handler"
3. Save

### Step 5: Configure Airtable Nodes in Error Handler

In the `04-error-handler` workflow, update these nodes with your Airtable IDs:

| Node Name | Base ID | Table ID |
|-----------|---------|----------|
| Lookup Project Recipients | Your Base ID | Projects table ID |
| Check Deduplication | Your Base ID | Error Deduplication table ID |
| Update Dedup Record | Your Base ID | Error Deduplication table ID |
| Create Dedup Record | Your Base ID | Error Deduplication table ID |
| Increment Dedup Count | Your Base ID | Error Deduplication table ID |

To find your Table IDs:
1. Open Airtable in your browser
2. Navigate to the table
3. The URL format is: `airtable.com/{BASE_ID}/{TABLE_ID}/...`

### Step 6: Activate Workflows

| Workflow | Activate? |
|----------|-----------|
| 01 - Notification Receiver | Yes |
| 02 - Deploy Notifications | No (sub-workflow) |
| 03 - Slack Interaction Handler | Yes |
| 04 - Error Handler | No (triggered by error) |

## Verification

### Test Happy Path
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
Expected: Slack notification appears in `#deployments` channel.

### Test Email Fallback
1. Temporarily use invalid Slack credentials in `02-deploy-notifications`
2. Send test webhook (as above)
3. Wait 4+ minutes for retries to exhaust
4. Verify:
   - Email sent to project recipients (from Airtable)
   - Admin alert sent to `don@callteksupport.com`
   - Deduplication record created in Airtable

### Test Deduplication
1. With invalid credentials still set, send another webhook within 5 minutes
2. Verify: No new email sent (check Airtable - count should increment)

## Troubleshooting

### Emails Not Sending

1. **Check SES Sender Verification**
   - AWS Console > SES > Verified Identities
   - Ensure sender email is verified

2. **Check SES Sandbox Mode**
   - New SES accounts are in sandbox mode
   - Request production access or verify recipient emails

3. **Check n8n Execution History**
   - n8n > Executions > Filter by 04-error-handler
   - Look for error messages in failed executions

### Airtable Lookups Failing

1. **Check Credentials**
   - n8n > Credentials > Test "Airtable - n8n Notifications"

2. **Check Base/Table IDs**
   - Ensure IDs in workflow match actual Airtable IDs

3. **Check API Token Permissions**
   - Token needs read/write access to the specific base

### Error Workflow Not Triggering

1. **Verify Error Workflow is Set**
   - Open each workflow > Settings > Error Workflow
   - Must show "04 - Error Handler"

2. **Check Workflow is Active**
   - Error handler should NOT be active (it's triggered)
   - Source workflows (01, 02, 03) must be active

### Retries Not Working

1. **Check continueOnFail Setting**
   - Slack/HTTP nodes must have `continueOnFail: true`

2. **Check IF Node Conditions**
   - Verify error detection logic in IF nodes

## Configuration Reference

### Retry Behavior
- **Retry Count**: 2 retries (3 attempts total)
- **Retry Interval**: 2 minutes between attempts
- **Total Window**: 4 minutes before email fallback

### Deduplication
- **Window**: 5 minutes
- **Key Format**: `{error_type}_{project_id}`
- **Behavior**: Admin alerts deduplicated; notification emails always sent

### Email Recipients
- **Notification Fallback**: Project-specific recipients from Airtable
- **Admin Alerts**: `don@callteksupport.com` (hardcoded)
- **Default Fallback**: If project not in Airtable, uses admin email

### Error Types Detected
- `auth_error` - Authentication/token issues
- `rate_limit_error` - API rate limiting
- `timeout_error` - Request timeouts
- `network_error` - Connection failures
- `slack_api_error` - Slack API errors
- `http_error` - General HTTP errors
- `unknown_error` - Unclassified errors

## Files Reference

| File | Description |
|------|-------------|
| `workflows/04-error-handler.json` | Error handler workflow (new) |
| `workflows/02-deploy-notifications.json` | Deploy workflow with retry logic |
| `workflows/03-slack-interactions.json` | Interactions workflow with retry logic |
| `docs/AIRTABLE-SETUP.md` | Airtable configuration guide |
| `docs/N8N-CREDENTIALS-SETUP.md` | n8n credentials setup guide |
