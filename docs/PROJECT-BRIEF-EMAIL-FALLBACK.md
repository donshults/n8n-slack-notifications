# Project Brief: Email Fallback for Slack Notification Failures

**Status:** Proposed
**Date:** 2024-12-21
**Author:** Claude Code

---

## Problem Statement

When Slack API errors occur (e.g., `token_expired`, rate limits, service outages), CI/CD notifications silently fail. Teams have no visibility into deployment events during Slack outages, creating a critical gap in the notification pipeline.

**Trigger:** Production incident on 2024-12-21 where Slack token expiration caused all deployment notifications to fail silently.

---

## Proposed Solution

Add email as a fallback notification channel that activates when Slack delivery fails. The system should:

1. Attempt Slack delivery first (primary channel)
2. On Slack failure, automatically send via email (backup channel)
3. Include the original notification content plus error context
4. Notify admins that Slack is failing so they can remediate

---

## Current Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  02-deploy-notifications.json                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Start → If Started/Success/Failed → Format Message          │
│                                           │                  │
│                                           ▼                  │
│                                    Send Slack (no fallback)  │
│                                           │                  │
│                                           ▼                  │
│                                    [END - or ERROR]          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Current Slack Nodes:**
- `Send Slack (Started)` - position [950, 100]
- `Send Slack (Success)` - position [950, 300]
- `Send Slack (Failed)` - position [950, 500]

All three nodes have no error handling - failures terminate the workflow.

---

## Proposed Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  02-deploy-notifications.json (updated)                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Start → If Started/Success/Failed → Format Message          │
│                                           │                  │
│                                           ▼                  │
│                              ┌─────────────────────┐         │
│                              │ Try Send Slack      │         │
│                              └──────────┬──────────┘         │
│                                         │                    │
│                              ┌──────────┴──────────┐         │
│                              │                     │         │
│                         [Success]              [Error]       │
│                              │                     │         │
│                              ▼                     ▼         │
│                           [END]          ┌─────────────────┐ │
│                                          │ Format Email    │ │
│                                          │ (include error) │ │
│                                          └────────┬────────┘ │
│                                                   │          │
│                                                   ▼          │
│                                          ┌─────────────────┐ │
│                                          │ Send Email      │ │
│                                          │ Fallback        │ │
│                                          └────────┬────────┘ │
│                                                   │          │
│                                                   ▼          │
│                                          ┌─────────────────┐ │
│                                          │ Alert: Slack    │ │
│                                          │ is Down (email) │ │
│                                          └─────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Technical Approach

### Option A: n8n Error Workflow (Recommended)

Use n8n's built-in Error Workflow feature:
- Create a dedicated `Error Handler` workflow
- Configure `02-deploy-notifications.json` to use this error workflow
- Error workflow receives full context and sends email

**Pros:**
- Clean separation of concerns
- Reusable across all workflows
- n8n native feature

**Cons:**
- Requires workflow-level configuration in n8n UI
- Error context may need parsing

### Option B: Inline Error Branch

Add error output handling directly in the deploy workflow:
- Enable "Continue on Error" for Slack nodes
- Add IF node to check for errors
- Branch to email on failure

**Pros:**
- Self-contained in single workflow
- Explicit flow visible in workflow editor

**Cons:**
- Duplicates error handling for each Slack node
- More complex workflow JSON

### Option C: Wrapper Sub-workflow

Create a `Send Notification` sub-workflow that:
- Accepts channel (slack/email) and message
- Tries Slack first
- Falls back to email on failure

**Pros:**
- Single point of notification logic
- Easy to add more fallback channels later

**Cons:**
- Requires refactoring existing workflows

---

## Recommendation

**Option A (Error Workflow)** is recommended because:
1. Minimal changes to existing workflows
2. Centralized error handling
3. Can alert on ALL workflow failures, not just Slack
4. Native n8n pattern

---

## Implementation Scope

### New Workflow: `04-error-handler.json`

```json
{
  "name": "Error Handler",
  "nodes": [
    {
      "name": "Error Trigger",
      "type": "n8n-nodes-base.errorTrigger"
    },
    {
      "name": "Format Error Email",
      "type": "n8n-nodes-base.code",
      "notes": "Extract workflow name, error message, original payload"
    },
    {
      "name": "Send Email - Notification Fallback",
      "type": "n8n-nodes-base.emailSend",
      "notes": "Send original notification content via email"
    },
    {
      "name": "Send Email - Admin Alert",
      "type": "n8n-nodes-base.emailSend",
      "notes": "Alert admin that Slack integration is failing"
    }
  ]
}
```

### Configuration Required

| Setting | Value |
|---------|-------|
| Email SMTP credentials | Required in n8n |
| Fallback recipient(s) | e.g., `devops@company.com` |
| Admin alert recipient | e.g., `don@callteksupport.com` |
| Rate limiting | Avoid email flood during outages |

### Changes to Existing Workflows

| Workflow | Change |
|----------|--------|
| `01-notification-receiver.json` | Set error workflow to `04-error-handler` |
| `02-deploy-notifications.json` | Set error workflow to `04-error-handler` |
| `03-slack-interactions.json` | Set error workflow to `04-error-handler` |

---

## Email Content Template

### Notification Fallback Email

```
Subject: [n8n Fallback] Deploy {status} - {project_name} to {environment}

---
SLACK DELIVERY FAILED - EMAIL FALLBACK
---

Original Notification:
{formatted_slack_message_as_plain_text}

---
Error Details:
Workflow: {workflow_name}
Error: {error_message}
Time: {timestamp}
---

This email was sent because Slack notification delivery failed.
Please check the Slack integration at https://nca-n8n.callteksupport.net
```

### Admin Alert Email

```
Subject: [n8n ALERT] Slack Integration Failing

The n8n Slack integration is experiencing errors.

Error: {error_message}
Workflow: {workflow_name}
Time: {timestamp}

Action Required:
1. Check Slack App token at https://api.slack.com/apps
2. Verify n8n Slack credentials
3. Check Slack service status at https://status.slack.com

n8n Admin: https://nca-n8n.callteksupport.net
```

---

## Decisions (Confirmed)

1. **Error Logging & Rate Limiting:**
   - Log ALL errors for evaluation
   - Limit email notifications to one per 5 minutes (dedupe by error type)

2. **Email Provider:** Amazon SES

3. **Recipient Configuration:** Airtable
   - Default recipient for fallback
   - Per-source recipient lookup (based on project/notification source)

4. **Retry Logic:**
   - Retry Slack at 2 minutes after initial failure
   - Retry Slack at 4 minutes after initial failure
   - Fall back to email only after both retries fail

---

## Success Criteria

- [ ] Slack failures trigger email fallback within 30 seconds
- [ ] Admin receives alert when Slack integration fails
- [ ] Email contains equivalent information to Slack message
- [ ] No duplicate emails for the same error within 5-minute window
- [ ] Error handler doesn't fail (has its own fallback or alerting)

---

## Files to Create/Modify

| File | Action |
|------|--------|
| `workflows/04-error-handler.json` | Create |
| `workflows/02-deploy-notifications.json` | Modify (add error workflow setting) |
| `docs/EMAIL-FALLBACK-SETUP.md` | Create (SMTP configuration guide) |
| `CLAUDE.md` | Update (document error handling) |

---

## Next Steps

1. Architect review of this brief
2. Decision on open questions above
3. Create `04-error-handler.json` workflow
4. Configure SMTP credentials in n8n
5. Test with simulated Slack failure
6. Update documentation
