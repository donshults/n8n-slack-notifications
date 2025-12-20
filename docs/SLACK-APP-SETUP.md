# Slack App Setup for n8n Notifications

This guide walks through creating a Slack App that n8n will use to send notifications and receive interactive responses.

## Overview

The Slack App needs:
1. **Bot Token** - For sending messages to channels
2. **Incoming Webhook** - Alternative simple message sending
3. **Interactivity** - To receive button clicks and slash commands

---

## Step 1: Create Slack App

1. Go to [api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App**
3. Choose **From scratch**
4. Enter:
   - **App Name:** `n8n Notifications` (or your preference)
   - **Workspace:** Select your workspace
5. Click **Create App**

---

## Step 2: Configure Bot Token Scopes

1. In your app settings, go to **OAuth & Permissions**
2. Scroll to **Scopes** → **Bot Token Scopes**
3. Add these scopes:

| Scope | Purpose |
|-------|---------|
| `chat:write` | Send messages to channels |
| `chat:write.public` | Send to channels the bot isn't in |
| `channels:read` | List public channels |
| `groups:read` | List private channels (if needed) |
| `reactions:write` | Add emoji reactions |
| `files:write` | Upload files (logs, screenshots) |
| `users:read` | Look up user info for mentions |

---

## Step 3: Install App to Workspace

1. Scroll to the top of **OAuth & Permissions**
2. Click **Install to Workspace**
3. Review permissions and click **Allow**
4. Copy the **Bot User OAuth Token** (starts with `xoxb-`)

Save this token - you'll add it to n8n as a credential.

---

## Step 4: Enable Interactivity (For Button Clicks)

1. Go to **Interactivity & Shortcuts**
2. Toggle **Interactivity** to **On**
3. Enter your **Request URL:**
   ```
   https://n8n.yourdomain.com/webhook/slack-interactions
   ```
   (We'll create this webhook in n8n)
4. Click **Save Changes**

---

## Step 5: Add Slash Commands (Optional)

If you want commands like `/deploy-status`:

1. Go to **Slash Commands**
2. Click **Create New Command**
3. Configure:
   - **Command:** `/deploy-status`
   - **Request URL:** `https://n8n.yourdomain.com/webhook/slack-commands`
   - **Short Description:** `Check deployment status`
   - **Usage Hint:** `[project-name]`
4. Click **Save**

Repeat for other commands:

| Command | Description |
|---------|-------------|
| `/deploy-status` | Check current deployment status |
| `/recent-deploys` | List recent deployments |
| `/rollback` | Initiate a rollback |

---

## Step 6: Create Incoming Webhook (Fallback)

For simple notifications without bot token:

1. Go to **Incoming Webhooks**
2. Toggle **Activate Incoming Webhooks** to **On**
3. Click **Add New Webhook to Workspace**
4. Select a default channel (e.g., `#deployments`)
5. Click **Allow**
6. Copy the **Webhook URL**

This can be used as a fallback or for simpler integrations.

---

## Step 7: Invite Bot to Channels

The bot can only post to channels it's a member of (unless using `chat:write.public`):

1. Go to each channel where you want notifications
2. Type `/invite @n8n Notifications`
3. Or click the channel name → **Integrations** → **Add apps**

Recommended channels to create:

| Channel | Purpose |
|---------|---------|
| `#deployments` | All deployment notifications |
| `#alerts` | Error and warning notifications |
| `#deployment-approvals` | Production approval requests |
| `#context-vault` | Project-specific channel |

---

## Credentials Summary

After setup, you should have:

| Credential | Example | Where to Use |
|------------|---------|--------------|
| Bot Token | `xoxb-123...` | n8n Slack credential |
| Webhook URL | `https://hooks.slack.com/services/...` | GitHub Actions fallback |
| Signing Secret | `abc123...` | n8n webhook validation |

### Get Signing Secret

1. Go to **Basic Information**
2. Scroll to **App Credentials**
3. Copy **Signing Secret**

The signing secret validates that requests to your n8n webhooks actually come from Slack.

---

## Add Credentials to n8n

1. In n8n, go to **Credentials** → **Add Credential**
2. Search for **Slack**
3. Choose **Slack OAuth2 API**
4. Enter your **Bot Token**
5. Save

For webhook validation:
1. Create a new **Header Auth** credential
2. Name: `Slack Signing Secret`
3. Use in webhook workflows to validate requests

---

## Test the Setup

### Test Bot Token

In n8n, create a simple workflow:
1. **Manual Trigger** → **Slack** node
2. Configure Slack node:
   - **Resource:** Message
   - **Operation:** Send
   - **Channel:** `#deployments`
   - **Text:** `Test message from n8n`
3. Execute and check Slack

### Test Webhook

1. Create workflow with **Webhook** trigger
2. Set path to `slack-interactions`
3. Activate workflow
4. In Slack app settings, re-save the Interactivity URL
5. Slack will send a verification request
6. Check n8n execution logs

---

## App Manifest (Alternative Setup)

You can also create the app from a manifest. Create the app with this YAML:

```yaml
display_information:
  name: n8n Notifications
  description: CI/CD and project notifications via n8n
  background_color: "#FF6D00"

features:
  bot_user:
    display_name: n8n Notifications
    always_online: true
  slash_commands:
    - command: /deploy-status
      url: https://n8n.yourdomain.com/webhook/slack-commands
      description: Check deployment status
      usage_hint: "[project-name]"
    - command: /recent-deploys
      url: https://n8n.yourdomain.com/webhook/slack-commands
      description: List recent deployments
      usage_hint: "[count]"

oauth_config:
  scopes:
    bot:
      - chat:write
      - chat:write.public
      - channels:read
      - groups:read
      - reactions:write
      - files:write
      - users:read

settings:
  interactivity:
    is_enabled: true
    request_url: https://n8n.yourdomain.com/webhook/slack-interactions
  org_deploy_enabled: false
  socket_mode_enabled: false
  token_rotation_enabled: false
```

To use:
1. Go to [api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App** → **From an app manifest**
3. Select workspace
4. Paste the YAML
5. Review and create

---

## Security Notes

1. **Never commit tokens to git** - Use environment variables or n8n credentials
2. **Validate webhook signatures** - Check Slack signing secret on incoming requests
3. **Restrict channels** - Bot only needs access to notification channels
4. **Rotate tokens periodically** - Update tokens every few months

---

## Next Steps

1. ✅ Slack App created
2. ✅ Bot token obtained
3. ✅ Interactivity enabled
4. → **Create n8n workflows** (see workflow JSON files)
5. → **Configure Airtable** for project settings
6. → **Update GitHub Actions** to use n8n
