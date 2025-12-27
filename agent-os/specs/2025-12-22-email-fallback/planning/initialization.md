# Spec Initialization: Email Fallback for Slack Notification Failures

## Source
Project brief: `/docs/PROJECT-BRIEF-EMAIL-FALLBACK.md`

## Raw Idea/Description

### Problem Statement

When Slack API errors occur (e.g., `token_expired`, rate limits, service outages), CI/CD notifications silently fail. Teams have no visibility into deployment events during Slack outages, creating a critical gap in the notification pipeline.

**Trigger:** Production incident on 2024-12-21 where Slack token expiration caused all deployment notifications to fail silently.

### Proposed Solution

Add email as a fallback notification channel that activates when Slack delivery fails. The system should:

1. Attempt Slack delivery first (primary channel)
2. On Slack failure, automatically send via email (backup channel)
3. Include the original notification content plus error context
4. Notify admins that Slack is failing so they can remediate

### Technical Approach (Recommended)

Use n8n's built-in Error Workflow feature:
- Create a dedicated `Error Handler` workflow (`04-error-handler.json`)
- Configure existing workflows to use this error workflow
- Error workflow receives full context and sends email

### Confirmed Decisions

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

### Success Criteria

- Slack failures trigger email fallback within 30 seconds
- Admin receives alert when Slack integration fails
- Email contains equivalent information to Slack message
- No duplicate emails for the same error within 5-minute window
- Error handler doesn't fail (has its own fallback or alerting)

### Files to Create/Modify

| File | Action |
|------|--------|
| `workflows/04-error-handler.json` | Create |
| `workflows/02-deploy-notifications.json` | Modify (add error workflow setting) |
| `docs/EMAIL-FALLBACK-SETUP.md` | Create (SMTP configuration guide) |
| `CLAUDE.md` | Update (document error handling) |

---

*Initialized: 2025-12-22*
