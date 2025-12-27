# Architectural Decisions: Email Fallback for Slack Notification Failures

**Spec:** 2025-12-22-email-fallback
**Date:** 2025-12-22
**Status:** Finalized

---

## Decision Summary

| # | Decision Area | Choice | Rationale |
|---|---------------|--------|-----------|
| 1 | Error Workflow Scope | All 3 workflows | Comprehensive coverage for all notification paths |
| 2 | Retry Implementation | Inline with Wait nodes | Simpler, self-contained, visible in workflow editor |
| 3 | Retry Intervals | 2 min, 2 min | Allows transient issues to resolve without excessive delay |
| 4 | Deduplication Key | Error type + Project | Prevents floods while allowing distinct issues to surface |
| 5 | Deduplication Storage | Airtable | Already used for config; avoids additional infrastructure |
| 6 | Notification Recipients | Project-specific (Airtable) | Enables per-project routing without code changes |
| 7 | Admin Alerts | don@callteksupport.com | Single admin contact for Slack integration issues |
| 8 | Email Format | HTML | Professional appearance, better readability |
| 9 | Email Failure Logging | n8n Execution History | Native solution, no additional tooling required |
| 10 | HTTP Slack Calls | Include in error handling | Ensures 03-slack-interactions.json failures are caught |
| 11 | Airtable Notes Column | Included | Allows documentation of project-specific configurations |
| 12 | Airtable Base | New base | Clean separation from any existing Airtable data |
| 13 | SES Configuration | Setup required | n8n credential needs to be created during implementation |

---

## Detailed Decisions

### 1. Error Workflow Scope

**Decision:** Error workflow supports all 3 existing workflows
- `01-notification-receiver.json`
- `02-deploy-notifications.json`
- `03-slack-interactions.json`

**Rationale:** Comprehensive error handling across the entire notification system ensures no silent failures regardless of which workflow encounters an issue.

---

### 2. Retry Implementation Approach

**Decision:** Option A - Inline Retry with Wait Nodes

**Pattern:**
```
Slack Node (continueOnFail: true)
  -> IF Error
    -> Wait 2min
    -> Retry Slack
    -> IF Error
      -> Wait 2min
      -> Retry Slack
      -> IF Error
        -> Trigger Error Workflow
```

**Alternatives Considered:**
- Option B: Separate retry sub-workflow - Rejected (adds complexity, harder to debug)

**Rationale:**
- Self-contained within each workflow
- Visible flow in n8n workflow editor
- Easier to understand and maintain
- No sub-workflow dependencies

---

### 3. Retry Timing

**Decision:** Two retries at 2-minute intervals

**Sequence:**
1. Initial attempt: immediate
2. First retry: 2 minutes after initial failure
3. Second retry: 2 minutes after first retry (4 minutes total from start)
4. Email fallback: triggered after second retry fails

**Rationale:**
- Transient Slack issues often resolve within 2-4 minutes
- Total 4-minute window balances reliability vs. notification delay
- Two retries provide reasonable persistence without excessive waiting

---

### 4. Deduplication Strategy

**Decision:** Deduplicate by both error type AND project

**Implementation:**
- 5-minute deduplication window
- Unique key: `{error_type}_{project_id}`
- State stored in Airtable

**Rationale:**
- Prevents email flood during extended Slack outages
- Still surfaces distinct issues per project
- Allows different error types to be reported separately

---

### 5. Deduplication State Storage

**Decision:** Airtable

**Table:** "Error Deduplication"
- Columns: error_type, project_id, last_sent, count

**Alternatives Considered:**
- Redis/in-memory - Rejected (requires additional infrastructure)
- n8n static data - Rejected (less reliable, harder to inspect)

**Rationale:**
- Airtable already being used for configuration
- Easy to inspect and manage
- Persistent across n8n restarts
- No additional infrastructure needed

---

### 6. Notification Recipient Configuration

**Decision:** Project-specific recipients from Airtable

**Table:** "Projects"
- Columns: project_id, project_name, recipients, notes

**Behavior:**
- Look up project_id in Airtable
- Send notification fallback to configured recipients
- If project not found, use default recipient

**Rationale:**
- Flexibility to route different projects to different teams
- Configuration changes don't require workflow edits
- Notes column allows documentation

---

### 7. Admin Alert Recipient

**Decision:** don@callteksupport.com

**Purpose:** Receives alerts about Slack integration failures (separate from notification fallbacks)

**Rationale:** Single point of contact for system-level issues that require remediation.

---

### 8. Email Format

**Decision:** HTML formatted emails

**Structure:**
- Header with status badge/styling
- Original notification content section
- Error details section
- Footer with remediation links

**Rationale:**
- Professional appearance
- Better readability with formatting
- Can include styling similar to Slack blocks
- Easier to scan for key information

---

### 9. Email Failure Handling

**Decision:** Log to n8n Execution History only

**Behavior:**
- If email sending fails, error is captured in execution history
- No secondary alerting mechanism
- Admin can review failed executions in n8n UI

**Alternatives Considered:**
- Secondary fallback (SMS, webhook) - Rejected (over-engineering for current needs)
- External logging service - Rejected (unnecessary complexity)

**Rationale:**
- Keeps error handler simple and less likely to fail
- n8n execution history is already available
- Avoids infinite error handling loops

---

### 10. HTTP-based Slack Calls

**Decision:** Include in error handling

**Scope:** HTTP Request nodes in `03-slack-interactions.json` that call Slack API will:
- Use continueOnFail
- Follow same retry pattern as Slack nodes
- Trigger error workflow on final failure

**Rationale:** Ensures consistent error handling across all Slack communication methods.

---

### 11. Airtable Notes Column

**Decision:** Include notes column in Projects table

**Purpose:**
- Document project-specific configurations
- Record contact information or escalation paths
- Store context about recipient choices

**Rationale:** Improves maintainability and reduces tribal knowledge.

---

### 12. Airtable Base

**Decision:** Create NEW Airtable base for this project

**Base Name:** "n8n Notifications"

**Tables:**
1. Projects - project configuration and recipients
2. Error Deduplication - rate limiting state

**Rationale:**
- Clean separation from any existing data
- Purpose-built schema for this use case
- Easier to manage permissions

---

### 13. SES Configuration

**Decision:** n8n SES credential setup required as part of implementation

**Status:** Amazon SES is available as AWS service, but n8n-specific configuration needed:
- Create AWS SES credential in n8n
- Configure AWS access key, secret key, region
- Verify sender email address in SES

**Rationale:** Enables email sending directly from n8n workflows without external dependencies.

---

## Implementation Notes

### Files to Create
- `workflows/04-error-handler.json` - Error handling workflow
- `docs/EMAIL-FALLBACK-SETUP.md` - Setup documentation

### Files to Modify
- `workflows/02-deploy-notifications.json` - Add retry logic to Slack nodes
- `workflows/03-slack-interactions.json` - Add retry logic to HTTP Request nodes
- `CLAUDE.md` - Document error handling architecture

### n8n UI Configuration Required
- Set error workflow on all 3 existing workflows
- Create SES credential
- Create Airtable credential (if not exists)

### Airtable Setup Required
- Create new base "n8n Notifications"
- Create Projects table with schema
- Create Error Deduplication table with schema
- Populate initial project configurations
