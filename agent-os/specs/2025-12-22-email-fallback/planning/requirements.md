# Spec Requirements: Email Fallback for Slack Notification Failures

## Initial Description

When Slack API errors occur (e.g., `token_expired`, rate limits, service outages), CI/CD notifications silently fail. Teams have no visibility into deployment events during Slack outages, creating a critical gap in the notification pipeline.

**Trigger:** Production incident on 2024-12-21 where Slack token expiration caused all deployment notifications to fail silently.

**Solution:** Add email as a fallback notification channel that activates when Slack delivery fails after retry attempts.

## Requirements Discussion

### First Round Questions

**Q1:** Should the error workflow be triggered by all 3 existing workflows, or only specific ones?
**Answer:** Error workflow supports all 3 workflows (01-notification-receiver.json, 02-deploy-notifications.json, 03-slack-interactions.json)

**Q2:** For retry logic, should we use inline Wait nodes within each workflow, or a separate retry sub-workflow?
**Answer:** Option A - Inline Retry with Wait Nodes. Pattern: Slack Node (continueOnFail) -> IF Error -> Wait 2min -> Retry -> IF Error -> Wait 2min -> Retry -> IF Error -> Trigger Error Workflow

**Q3:** Should deduplication be by error type only, or by error type AND project?
**Answer:** Deduplication by both error type AND project

**Q4:** Where should deduplication state be stored?
**Answer:** Airtable

**Q5:** For notification fallback emails, should they go to project-specific recipients or a default recipient?
**Answer:** Notification fallback emails go to project recipients (looked up from Airtable). Admin alerts go to don@callteksupport.com

**Q6:** Should emails be plain text or HTML formatted?
**Answer:** HTML formatted emails

**Q7:** How should email sending failures be logged/handled?
**Answer:** n8n Execution History only for email failure logging (no additional logging mechanism)

**Q8:** The 03-slack-interactions workflow uses HTTP Request nodes for Slack API calls. Should these also trigger the error workflow on failure?
**Answer:** Yes, HTTP-based Slack calls in 03-slack-interactions.json should also trigger the error workflow

**Q9:** Should the Airtable configuration table include a notes/description column for each project?
**Answer:** Yes, Airtable table includes notes column

**Q10:** Is there an existing Airtable base to use, or should we create a new one?
**Answer:** Create a NEW Airtable base for this project

**Q11:** Is Amazon SES already configured as an n8n credential, or does that need to be set up?
**Answer:** Amazon SES is already set up as an AWS service, but n8n credential/node configuration needs to be done as part of this feature

### Existing Code to Reference

**Similar Features Identified:**
- Workflow: `02-deploy-notifications.json` - Contains Slack notification nodes that will need retry logic added
- Workflow: `03-slack-interactions.json` - Contains HTTP Request nodes for Slack API that need error handling
- Workflow: `01-notification-receiver.json` - Main entry point that routes to other workflows

**Components to potentially reuse:**
- Existing Slack message formatting in 02-deploy-notifications.json
- Channel routing configuration in 01-notification-receiver.json

**Backend logic to reference:**
- Validation & Configure Code node pattern in 01-notification-receiver.json

### Follow-up Questions

**Follow-up 1:** Confirming retry implementation approach - Option A (Inline with Wait nodes) or Option B (Separate retry workflow)?
**Answer:** Option A - Inline Retry with Wait Nodes (NOT Option B). Retry logic will be inline within each workflow using Wait nodes.

**Follow-up 2:** Which Airtable base should be used for configuration?
**Answer:** Create a NEW Airtable base for this project

**Follow-up 3:** What is the admin email address for alerts?
**Answer:** don@callteksupport.com

**Follow-up 4:** Does SES credential need to be configured in n8n?
**Answer:** Yes, n8n SES credential setup is required as part of this feature

## Visual Assets

### Files Provided:
No visual assets provided.

### Visual Insights:
N/A

## Requirements Summary

### Functional Requirements

**Error Detection & Retry:**
- All Slack nodes configured with `continueOnFail: true`
- IF node checks for error after each Slack call
- On error: Wait 2 minutes, then retry
- On second error: Wait 2 minutes, then retry again
- On third error: Trigger error workflow
- HTTP Request nodes in 03-slack-interactions.json also follow this pattern

**Email Fallback:**
- Error workflow sends notification content via email when Slack fails
- Project-specific recipients looked up from Airtable
- HTML formatted emails with equivalent content to Slack messages
- Email includes error context (workflow name, error message, timestamp)

**Admin Alerting:**
- Separate admin alert email sent to don@callteksupport.com
- Alert indicates Slack integration is failing
- Includes remediation steps

**Deduplication:**
- Rate limit emails to one per 5 minutes
- Deduplicate by both error type AND project
- Deduplication state stored in Airtable
- Prevents email flood during extended outages

**Configuration Management:**
- New Airtable base created for this project
- Tables include:
  - Project configuration (project ID, recipients, notes)
  - Deduplication state (error type, project, last sent timestamp)
- n8n SES credential configuration required

### Reusability Opportunities

- Error workflow pattern can be applied to future workflows
- Retry logic pattern (Wait node approach) can be templated
- Airtable configuration lookup can be extended for other settings
- HTML email templates can be reused for other notification types

### Scope Boundaries

**In Scope:**
- Create 04-error-handler.json workflow
- Add inline retry logic to 02-deploy-notifications.json (all Slack nodes)
- Add inline retry logic to 03-slack-interactions.json (HTTP Request nodes)
- Configure error workflow setting on all 3 existing workflows
- Set up Airtable base with project configuration and deduplication tables
- Configure n8n SES credential
- Create email templates (notification fallback, admin alert)
- Create EMAIL-FALLBACK-SETUP.md documentation
- Update CLAUDE.md with error handling documentation

**Out of Scope:**
- SMS or other notification channels
- Automatic Slack token refresh
- UI for managing Airtable configuration
- Retry logic for 01-notification-receiver.json (routes only, no Slack calls)
- Custom logging system beyond n8n execution history

### Technical Considerations

**n8n Configuration:**
- Error workflow must be set at workflow level in n8n UI for each workflow
- Wait nodes use n8n's native wait functionality (2-minute intervals)
- continueOnFail must be enabled on all Slack nodes
- SES node requires AWS credentials (access key, secret key, region)

**Airtable Schema:**
- Base: "n8n Notifications" (new)
- Table 1: "Projects" - columns: project_id, project_name, recipients, notes
- Table 2: "Error Deduplication" - columns: error_type, project_id, last_sent, count

**Email Requirements:**
- HTML format with clear sections
- Subject line includes project name and status
- Body includes original notification content
- Error details section with workflow name, error message, timestamp
- Footer with link to n8n admin panel

**Integration Points:**
- Amazon SES (email sending)
- Airtable (configuration and deduplication state)
- n8n Error Trigger node (captures workflow failures)
- Existing Slack nodes (modified with retry logic)

**Retry Timing:**
- Initial attempt: immediate
- First retry: 2 minutes after initial failure
- Second retry: 2 minutes after first retry (4 minutes total)
- Email fallback: after second retry fails

**Success Criteria:**
- Slack failures trigger email fallback within 30 seconds (after retry sequence completes)
- Admin receives alert when Slack integration fails
- Email contains equivalent information to Slack message
- No duplicate emails for the same error within 5-minute window
- Error handler doesn't fail (logs to execution history on email failure)
