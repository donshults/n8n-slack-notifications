# Specification: Email Fallback for Slack Notification Failures

## Goal

Add email as a fallback notification channel that activates when Slack delivery fails after retry attempts, ensuring teams maintain visibility into CI/CD events during Slack outages.

## User Stories

- As a DevOps engineer, I want to receive deployment notifications via email when Slack is unavailable so that I never miss critical deployment events.
- As a system administrator, I want to be alerted when the Slack integration is failing so that I can remediate the issue promptly.

## Specific Requirements

**Inline Retry Logic for Slack Nodes**
- All Slack nodes in `02-deploy-notifications.json` must have `continueOnFail: true` enabled
- After each Slack node, add IF node to check for error condition
- On error: Wait 2 minutes using n8n Wait node, then retry
- On second error: Wait 2 minutes, then retry again
- On third error: Trigger error workflow via n8n Error Trigger mechanism
- Total retry window is 4 minutes before fallback activates

**Inline Retry Logic for HTTP Request Nodes**
- All HTTP Request nodes in `03-slack-interactions.json` that call Slack API must have `continueOnFail: true`
- Apply same retry pattern: IF Error -> Wait 2min -> Retry -> IF Error -> Wait 2min -> Retry -> IF Error -> Trigger Error Workflow
- Applies to: "Update Message - Approved", "Update Message - Rejected", "Update Message - Rollback" nodes

**Error Handler Workflow (04-error-handler.json)**
- New workflow triggered by n8n Error Trigger node
- Receives full error context including: workflow name, error message, original payload, timestamp
- Performs Airtable lookup for project recipients and deduplication check
- Sends notification fallback email to project recipients if within deduplication window
- Sends admin alert email to don@callteksupport.com
- Must not throw errors itself (use try-catch patterns in Code nodes)

**Email Deduplication**
- Rate limit: one email per 5 minutes per unique combination of error type AND project
- Deduplication key format: `{error_type}_{project_id}`
- Before sending email, query Airtable "Error Deduplication" table for matching record
- If last_sent timestamp is within 5 minutes, skip sending and increment count
- If outside window or no record exists, send email and update/create record

**Project Recipient Lookup**
- Query Airtable "Projects" table using project_id from error context
- If project found, send notification fallback email to configured recipients
- If project not found, use don@callteksupport.com as default recipient
- Recipients field supports multiple comma-separated email addresses

**HTML Email Templates**
- Notification fallback email includes: status badge, original notification content, error details section, footer with n8n admin link
- Admin alert email includes: error summary, affected workflow, recommended remediation steps, links to Slack status and n8n admin
- Subject lines must include project name and status for easy filtering
- Use inline CSS for email client compatibility

**Amazon SES Integration**
- Configure AWS SES credential in n8n with access key, secret key, and region
- Use n8n SES node for sending emails
- Sender address must be verified in SES console
- Email failures logged to n8n Execution History only (no secondary fallback)

**Workflow Error Workflow Configuration**
- All 3 existing workflows must have error workflow setting configured in n8n UI
- Error workflow setting points to `04-error-handler.json` workflow ID
- Configuration done manually in n8n UI after workflow import

## Visual Design

No visual assets provided.

## Existing Code to Leverage

**02-deploy-notifications.json - Slack Notification Nodes**
- Contains 3 Slack nodes at positions [950, 100/300/500]: "Send Slack (Started)", "Send Slack (Success)", "Send Slack (Failed)"
- Each currently has no error handling; failures terminate workflow
- Formatting code in "Format Started/Success/Failed" Code nodes can be adapted for email HTML generation
- Message content structure provides template for email body content

**03-slack-interactions.json - HTTP Request Nodes**
- Contains 3 HTTP Request nodes for Slack response_url updates: "Update Message - Approved", "Update Message - Rejected", "Update Message - Rollback"
- Currently no error handling on these nodes
- Response URL pattern and message update structure informs retry logic design

**01-notification-receiver.json - Validation Pattern**
- "Validate & Configure" Code node demonstrates pattern for data validation and configuration lookup
- defaultConfig object pattern can be replicated for error handler configuration
- Channel routing logic informs how recipient lookup should work

**Existing Workflow Structure**
- All workflows use n8n workflow JSON format with nodes, connections, and settings objects
- Node positioning conventions: x-axis indicates flow progression, y-axis indicates parallel branches
- Settings include `executionOrder: "v1"` which must be preserved

**Event Payload Schema**
- Standard payload includes: event_type, status, project.id, project.name, environment, commit details, actor info
- Error context must preserve these fields for email content generation

## Out of Scope

- SMS or other notification channels beyond email
- Automatic Slack token refresh or rotation
- UI for managing Airtable configuration (manual Airtable editing only)
- Retry logic for 01-notification-receiver.json (routes events only, no direct Slack calls)
- Custom logging system beyond n8n execution history
- Email delivery confirmation or read receipts
- Scheduled recovery checks for Slack integration
- Integration with PagerDuty or other incident management systems
- Slack API health monitoring dashboard
- Automatic failover between multiple email providers
