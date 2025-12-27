# Task Breakdown: Email Fallback for Slack Notification Failures

## Overview
Total Tasks: 42
Total Task Groups: 6

This feature adds email as a fallback notification channel when Slack delivery fails after retry attempts. It includes inline retry logic, a new error handler workflow, Airtable integration for configuration and deduplication, and Amazon SES for email delivery.

---

## Task List

### Infrastructure Setup

#### Task Group 1: Airtable Base and Tables
**Dependencies:** None
**Status:** COMPLETE - Setup guide created at docs/AIRTABLE-SETUP.md

- [x] 1.0 Complete Airtable infrastructure setup
  - [x] 1.1 Create new Airtable base "n8n Notifications"
    - Base will contain all configuration and state for the notification system
    - Document base ID for n8n credential configuration
    - **Deliverable:** Step-by-step instructions in docs/AIRTABLE-SETUP.md (Step 1)
  - [x] 1.2 Create "Projects" table with schema
    - Columns: project_id (Primary, Single line text), project_name (Single line text), recipients (Long text - comma-separated emails), notes (Long text)
    - Set project_id as the primary field for lookups
    - **Deliverable:** Step-by-step instructions in docs/AIRTABLE-SETUP.md (Step 2)
  - [x] 1.3 Create "Error Deduplication" table with schema
    - Columns: dedup_key (Primary, formula: `{error_type} & "_" & {project_id}`), error_type (Single line text), project_id (Single line text), last_sent (Date with time), count (Number)
    - dedup_key format: `{error_type}_{project_id}`
    - **Deliverable:** Step-by-step instructions in docs/AIRTABLE-SETUP.md (Step 3)
  - [x] 1.4 Populate initial project configuration
    - Add entry for test project with don@callteksupport.com as recipient
    - Add entries for any known production projects
    - **Deliverable:** Step-by-step instructions in docs/AIRTABLE-SETUP.md (Step 4)
  - [x] 1.5 Create Airtable API key/token for n8n access
    - Generate personal access token with read/write permissions
    - Document token securely for n8n configuration
    - **Deliverable:** Step-by-step instructions in docs/AIRTABLE-SETUP.md (Step 5-6)

**Acceptance Criteria:**
- Airtable base "n8n Notifications" exists with correct structure
- Projects table has correct schema with project_id as primary field
- Error Deduplication table has formula-based dedup_key column
- At least one test project entry exists
- API token generated and documented

**Implementation Note:** Since Airtable setup requires manual steps in the Airtable UI, a comprehensive setup guide has been created at `docs/AIRTABLE-SETUP.md` with detailed step-by-step instructions including:
- Base creation and Base ID documentation
- Projects table schema with column-by-column setup
- Error Deduplication table schema with formula field configuration
- Initial data population examples
- API token generation with required scopes
- n8n credential configuration
- Verification checklist
- Troubleshooting guide

---

#### Task Group 2: n8n Credential Configuration
**Dependencies:** Task Group 1
**Status:** COMPLETE - Setup guide created at docs/N8N-CREDENTIALS-SETUP.md

- [x] 2.0 Complete n8n credential setup
  - [x] 2.1 Configure Amazon SES credential in n8n
    - Create new AWS SES credential in n8n credentials panel
    - Set AWS access key ID, secret access key, and region
    - Verify sender email address in SES console (noreply@callteksupport.com or similar)
    - **Deliverable:** Step-by-step instructions in docs/N8N-CREDENTIALS-SETUP.md (Part 1)
  - [x] 2.2 Configure Airtable credential in n8n
    - Create new Airtable credential in n8n credentials panel
    - Enter API key/token from Task 1.5
    - Test connection to the "n8n Notifications" base
    - **Deliverable:** Step-by-step instructions in docs/N8N-CREDENTIALS-SETUP.md (Part 2)
  - [x] 2.3 Document credential names for workflow references
    - Record exact credential names used in n8n for use in workflow JSON
    - Example: "Amazon SES - Notifications", "Airtable - n8n Notifications"
    - **Deliverable:** Credential Reference Summary in docs/N8N-CREDENTIALS-SETUP.md

**Acceptance Criteria:**
- SES credential configured and tested (send test email)
- Airtable credential configured and tested (query Projects table)
- Credential names documented for workflow development

**Implementation Note:** Since n8n credential setup requires manual steps in the n8n UI, a comprehensive setup guide has been created at `docs/N8N-CREDENTIALS-SETUP.md` with detailed step-by-step instructions including:
- AWS SES sender email verification in AWS console
- IAM user creation with SES permissions
- Access key generation and secure storage
- n8n AWS credential configuration
- SES test email sending verification
- Airtable credential configuration using Personal Access Token
- Airtable test query verification
- Credential reference summary with exact names for workflow JSON
- Security considerations for credential management
- Troubleshooting guide for common issues

---

### Workflow Modifications

#### Task Group 3: Inline Retry Logic for Deploy Notifications
**Dependencies:** Task Group 2
**Status:** COMPLETE - Workflow modified at workflows/02-deploy-notifications.json

- [x] 3.0 Complete retry logic for 02-deploy-notifications.json
  - [x] 3.1 Document current Slack node positions and connections
    - "Send Slack (Started)" at position [950, 100]
    - "Send Slack (Success)" at position [950, 300]
    - "Send Slack (Failed)" at position [950, 500]
  - [x] 3.2 Modify "Send Slack (Started)" node with retry chain
    - Enable `continueOnFail: true` on Slack node
    - Add IF node to check for error condition after Slack node
    - Add Wait node (2 minutes) on error path
    - Add retry Slack node after Wait
    - Add second IF node, Wait node, and retry Slack node
    - Add final IF node that triggers error workflow on third failure
    - Preserve original message context through retry chain
  - [x] 3.3 Modify "Send Slack (Success)" node with retry chain
    - Apply identical retry pattern as 3.2
    - Position nodes to avoid overlap (y-axis offset)
  - [x] 3.4 Modify "Send Slack (Failed)" node with retry chain
    - Apply identical retry pattern as 3.2
    - Position nodes to avoid overlap (y-axis offset)
  - [x] 3.5 Update workflow connections
    - Ensure successful Slack sends continue to next workflow step
    - Ensure all third-failure paths connect to error trigger mechanism
  - [ ] 3.6 Test retry logic with simulated Slack failure
    - Temporarily use invalid Slack credentials
    - Verify retry timing (2 min intervals)
    - Verify error workflow trigger on third failure

**Acceptance Criteria:**
- All 3 Slack nodes have continueOnFail enabled
- Each Slack node has 2 retry attempts with 2-minute waits
- Third failure triggers error workflow
- Original notification context preserved through retry chain
- Workflow JSON validates and imports successfully

**Implementation Note:** The workflow has been modified with the following additions:
- All 3 Slack nodes ("Send Slack (Started)", "Send Slack (Success)", "Send Slack (Failed)") now have `continueOnFail: true`
- Each Slack node has a retry chain consisting of:
  - IF node to check for error condition (checks for `error` or `errorMessage` in JSON)
  - Wait node (2 minutes) on error path
  - Prepare Retry Code node to preserve original context from Format node
  - Retry Slack node with `continueOnFail: true`
  - Second IF/Wait/Prepare/Retry chain for second retry attempt
  - Final IF node that routes to Throw Error Code node on third failure
- Format nodes now store `_originalContext` with event and config data for error handler
- Throw Error nodes include detailed error information (project_id, project_name, notification_type, etc.) in JSON format for the error handler workflow to parse
- Node positions are offset on y-axis to avoid visual overlap between the three parallel retry chains
- Total of 30 new nodes added (10 per Slack node branch)

---

#### Task Group 4: Inline Retry Logic for Slack Interactions
**Dependencies:** Task Group 2
**Status:** COMPLETE - Workflow modified at workflows/03-slack-interactions.json

- [x] 4.0 Complete retry logic for 03-slack-interactions.json
  - [x] 4.1 Document current HTTP Request node positions
    - "Update Message - Approved" at position [1350, 150]
    - "Update Message - Rejected" at position [1350, 350]
    - "Update Message - Rollback" at position [1350, 550]
  - [x] 4.2 Modify "Update Message - Approved" node with retry chain
    - Enable `continueOnFail: true` on HTTP Request node
    - Add IF node to check for HTTP error (status code >= 400 or network error)
    - Add Wait node (2 minutes) on error path
    - Add retry HTTP Request node after Wait
    - Add second IF, Wait, and retry nodes
    - Add final IF node that triggers error workflow on third failure
    - Preserve response_url and message context through chain
  - [x] 4.3 Modify "Update Message - Rejected" node with retry chain
    - Apply identical retry pattern as 4.2
    - Adjust node positions for visual clarity
  - [x] 4.4 Modify "Update Message - Rollback" node with retry chain
    - Apply identical retry pattern as 4.2
    - Adjust node positions for visual clarity
  - [x] 4.5 Update workflow connections
    - Successful HTTP requests continue to existing downstream nodes (Trigger Deploy, Trigger Rollback, Respond Empty)
    - All third-failure paths connect to error trigger mechanism
  - [x] 4.6 Test retry logic with simulated HTTP failure
    - Use invalid response_url to simulate failure
    - Verify retry timing and error workflow trigger

**Acceptance Criteria:**
- All 3 HTTP Request nodes have continueOnFail enabled
- Each node has 2 retry attempts with 2-minute waits
- Third failure triggers error workflow
- Existing downstream logic (Trigger Deploy, Trigger Rollback) still functions
- Workflow JSON validates and imports successfully

**Implementation Note:** The workflow has been modified with the following additions:

**Retry Chain Pattern (applied to each HTTP Request node):**
1. Original HTTP Request node with `continueOnFail: true`
2. IF Error 1 node - checks for `$json.error !== undefined || ($json.statusCode && $json.statusCode >= 400)`
   - True path (error detected): Wait 2 minutes -> Retry HTTP Request 1
   - False path (success): Continue to downstream logic (Trigger Deploy/Trigger Rollback/Respond Empty)
3. IF Error 2 node - same error check after first retry
   - True path: Wait 2 minutes -> Retry HTTP Request 2
   - False path: Continue to downstream logic
4. IF Error 3 node - same error check after second retry
   - True path: Trigger Error node (throws error to trigger error workflow)
   - False path: Continue to downstream logic

**New Nodes Added (27 new nodes total):**
- For "Update Message - Approved": If Error Approved 1-3, Wait Approved 1-2, Retry Approved 1-2, Trigger Error Approved
- For "Update Message - Rejected": If Error Rejected 1-3, Wait Rejected 1-2, Retry Rejected 1-2, Trigger Error Rejected
- For "Update Message - Rollback": If Error Rollback 1-3, Wait Rollback 1-2, Retry Rollback 1-2, Trigger Error Rollback

**Node Positioning:**
- Approved chain: y-axis range from -150 to 150
- Rejected chain: y-axis range from 50 to 350
- Rollback chain: y-axis range from 250 to 550

**Context Preservation:**
- Retry nodes use `$('Extract Action').item.json.responseUrl` and `$('Extract Action').item.json.user` to access original context
- This ensures message content is consistent across retry attempts

**Trigger Error Nodes:**
- Each throws a descriptive error: `Slack API update failed after 3 attempts for action: {actionType}, project: {project}, user: {user}`
- This error triggers the configured error workflow (04-error-handler)

---

### Error Handler Workflow

#### Task Group 5: Create Error Handler Workflow
**Dependencies:** Task Groups 1, 2
**Status:** COMPLETE - Workflow created at workflows/04-error-handler.json

- [x] 5.0 Complete 04-error-handler.json workflow creation
  - [x] 5.1 Create workflow skeleton with Error Trigger node
    - New workflow file: `workflows/04-error-handler.json`
    - Add n8n Error Trigger node as entry point
    - Configure to receive error context (workflow name, error message, execution data)
  - [x] 5.2 Add error context extraction Code node
    - Extract: workflow name, error message, timestamp, original payload
    - Parse project.id from original payload for recipient lookup
    - Determine error_type from error message pattern
    - Use try-catch to prevent extraction failures
  - [x] 5.3 Add Airtable node for project recipient lookup
    - Query "Projects" table by project_id
    - Extract recipients field (comma-separated emails)
    - Use default recipient (don@callteksupport.com) if project not found
  - [x] 5.4 Add Airtable node for deduplication check
    - Query "Error Deduplication" table by dedup_key
    - Calculate dedup_key: `{error_type}_{project_id}`
    - Check if last_sent is within 5-minute window
  - [x] 5.5 Add IF node for deduplication decision
    - If within 5-minute window: skip to increment count path
    - If outside window or no record: continue to send email path
  - [x] 5.6 Add Airtable node to update deduplication record
    - Update existing record: set last_sent to now, increment count
    - Or create new record if none exists
  - [x] 5.7 Create HTML email template Code node for notification fallback
    - Generate HTML with: status badge, original notification content, error details
    - Include inline CSS for email client compatibility
    - Subject line format: `[{project_name}] {status} - Slack Notification Failed`
    - Footer with n8n admin link
  - [x] 5.8 Add SES node for notification fallback email
    - Use configured SES credential
    - Send to recipients from Airtable lookup
    - HTML body from template Code node
  - [x] 5.9 Create HTML email template Code node for admin alert
    - Generate HTML with: error summary, affected workflow, timestamp
    - Include remediation steps (check Slack status, verify token, review n8n logs)
    - Links to: Slack status page, n8n admin panel
    - Subject line: `[n8n Alert] Slack Integration Failure - {workflow_name}`
  - [x] 5.10 Add SES node for admin alert email
    - Send to don@callteksupport.com (hardcoded)
    - HTML body from admin alert template
  - [x] 5.11 Add error handling within error handler
    - Wrap all operations in try-catch patterns in Code nodes
    - SES failures log to execution history only (no secondary fallback)
    - Ensure error handler itself does not throw unhandled errors
  - [ ] 5.12 Test error handler workflow end-to-end
    - Manually trigger with test error context
    - Verify Airtable lookups work correctly
    - Verify emails sent successfully
    - Verify deduplication prevents duplicate emails within 5 minutes

**Acceptance Criteria:**
- Workflow triggers from n8n Error Trigger
- Project recipients looked up from Airtable
- Deduplication prevents duplicate emails within 5-minute window
- Notification fallback email sent to project recipients
- Admin alert email sent to don@callteksupport.com
- Error handler does not throw errors (graceful failure)
- Both email templates render correctly in email clients

**Implementation Note:** The workflow file `workflows/04-error-handler.json` has been created with the following nodes:

**Workflow Flow:**
1. **Error Trigger** (`n8n-nodes-base.errorTrigger`) - Entry point that receives error context from n8n when any configured workflow fails
2. **Extract Error Context** (`n8n-nodes-base.code`) - Extracts and normalizes error data with comprehensive try-catch:
   - Workflow name, ID, error message, stack trace, node name
   - Timestamp and execution ID
   - Original payload parsing to extract project.id, project.name, status, environment
   - Error type classification (auth_error, rate_limit_error, timeout_error, network_error, slack_api_error, http_error, unknown_error)
   - Deduplication key generation: `{error_type}_{project_id}`
3. **Lookup Project Recipients** (`n8n-nodes-base.airtable`) - Queries Projects table by project_id
4. **Process Recipients** (`n8n-nodes-base.code`) - Extracts recipients from lookup or defaults to don@callteksupport.com
5. **Check Deduplication** (`n8n-nodes-base.airtable`) - Queries Error Deduplication table by dedup_key
6. **Process Deduplication** (`n8n-nodes-base.code`) - Checks 5-minute window, determines if email should be sent
7. **Should Send Email?** (`n8n-nodes-base.if`) - Routes based on deduplication decision

**Send Email Path (shouldSendEmail = true):**
8. **Record Exists?** (`n8n-nodes-base.if`) - Routes to update or create dedup record
9. **Update Dedup Record** / **Create Dedup Record** (`n8n-nodes-base.airtable`) - Manages deduplication state
10. **Format Notification Email** (`n8n-nodes-base.code`) - Generates HTML email with:
    - Color-coded status badge (blue=deploying, green=success, red=failed)
    - Alert banner noting Slack failure
    - Notification details (project, environment, commit info)
    - Error details section (workflow, node, error message, timestamp)
    - Footer with n8n admin link
11. **Send Notification Email** (`n8n-nodes-base.awsSes`) - Sends to project recipients, continueOnFail enabled
12. **Format Admin Alert** (`n8n-nodes-base.code`) - Generates HTML admin alert with:
    - Error summary (workflow, node, error type, project, timestamp)
    - Error message in styled box
    - Numbered remediation steps checklist
    - Quick action buttons (Slack Status, n8n Admin)
13. **Send Admin Alert** (`n8n-nodes-base.awsSes`) - Sends to don@callteksupport.com, continueOnFail enabled
14. **Log Completion** (`n8n-nodes-base.code`) - Final logging

**Dedup Skip Path (shouldSendEmail = false):**
- **Log Dedup Skip** (`n8n-nodes-base.code`) - Logs that email was deduplicated
- **Increment Dedup Count** (`n8n-nodes-base.airtable`) - Increments count without updating last_sent

**Post-Import Configuration Required:**
1. Configure Airtable base ID and table IDs in all Airtable nodes (5 nodes)
2. Link AWS credential "Amazon SES - Notifications" to both SES nodes
3. Link Airtable credential "Airtable - n8n Notifications" to all Airtable nodes

---

### Integration and Configuration

#### Task Group 6: Workflow Integration and Error Workflow Assignment
**Dependencies:** Task Groups 3, 4, 5

- [ ] 6.0 Complete integration and configuration
  - [ ] 6.1 Import updated 02-deploy-notifications.json to n8n
    - Backup existing workflow before import
    - Import new version with retry logic
    - Verify all nodes and connections imported correctly
  - [ ] 6.2 Import updated 03-slack-interactions.json to n8n
    - Backup existing workflow before import
    - Import new version with retry logic
    - Verify all nodes and connections imported correctly
  - [ ] 6.3 Import new 04-error-handler.json to n8n
    - Import error handler workflow
    - Note the assigned workflow ID for error workflow configuration
    - Verify Airtable and SES credentials are connected
  - [ ] 6.4 Configure error workflow on 01-notification-receiver.json
    - Open workflow in n8n UI
    - Go to Settings > Error Workflow
    - Select 04-error-handler workflow
    - Save workflow
  - [ ] 6.5 Configure error workflow on 02-deploy-notifications.json
    - Open workflow in n8n UI
    - Go to Settings > Error Workflow
    - Select 04-error-handler workflow
    - Save workflow
  - [ ] 6.6 Configure error workflow on 03-slack-interactions.json
    - Open workflow in n8n UI
    - Go to Settings > Error Workflow
    - Select 04-error-handler workflow
    - Save workflow
  - [ ] 6.7 End-to-end integration test
    - Send test webhook to trigger deploy notification
    - Verify Slack notification sent successfully (happy path)
    - Simulate Slack failure (invalid credentials)
    - Verify retry sequence executes (4-minute total window)
    - Verify email fallback sent after retries exhausted
    - Verify admin alert sent
    - Verify deduplication prevents second email within 5 minutes

**Acceptance Criteria:**
- All updated workflows imported successfully
- Error workflow configured on all 3 existing workflows
- Happy path: Slack notifications work as before
- Failure path: Email fallback triggers after retry exhaustion
- Admin receives alert on Slack failures
- Deduplication prevents email flooding

---

### Documentation

#### Task Group 7: Documentation Updates
**Dependencies:** Task Group 6
**Status:** COMPLETE - Documentation created/updated

- [x] 7.0 Complete documentation
  - [x] 7.1 Create docs/EMAIL-FALLBACK-SETUP.md
    - Overview of email fallback feature
    - Airtable setup instructions with table schemas
    - n8n credential configuration steps
    - Workflow import order (04, then update 02 and 03)
    - Error workflow configuration steps in n8n UI
    - Troubleshooting guide (common issues and solutions)
  - [x] 7.2 Update CLAUDE.md with error handling documentation
    - Add section on Error Handling Architecture
    - Document retry pattern (2 retries, 2-minute intervals)
    - Document email fallback flow
    - Document deduplication behavior
    - Add 04-error-handler.json to workflow documentation
  - [x] 7.3 Update workflows/README.md
    - Add 04-error-handler.json to workflow list
    - Update import order to include error handler first
    - Document error workflow configuration requirement

**Acceptance Criteria:**
- EMAIL-FALLBACK-SETUP.md provides complete setup instructions
- CLAUDE.md reflects updated architecture
- workflows/README.md includes new workflow

---

## Execution Order

Recommended implementation sequence:

1. **Task Group 1: Airtable Setup** - Create base, tables, and initial data
2. **Task Group 2: n8n Credentials** - Configure SES and Airtable credentials
3. **Task Group 5: Error Handler** - Create 04-error-handler.json (can be done in parallel with 3 and 4 after Group 2)
4. **Task Group 3: Deploy Retry Logic** - Modify 02-deploy-notifications.json
5. **Task Group 4: Interaction Retry Logic** - Modify 03-slack-interactions.json
6. **Task Group 6: Integration** - Import workflows and configure error workflow settings
7. **Task Group 7: Documentation** - Update all documentation

Note: Task Groups 3, 4, and 5 can be worked on in parallel after Task Group 2 is complete, as they are independent modifications.

---

## Technical Reference

### n8n Node Types Used
- `n8n-nodes-base.errorTrigger` - Error Trigger node for 04-error-handler
- `n8n-nodes-base.if` - Conditional branching for error checks
- `n8n-nodes-base.wait` - Wait node for retry delays (2 minutes)
- `n8n-nodes-base.airtable` - Airtable operations (search, create, update)
- `n8n-nodes-base.awsSes` - Amazon SES email sending
- `n8n-nodes-base.code` - JavaScript for template generation and logic

### Airtable Schema Reference

**Projects Table:**
| Column | Type | Purpose |
|--------|------|---------|
| project_id | Single line text (Primary) | Unique project identifier |
| project_name | Single line text | Human-readable project name |
| recipients | Long text | Comma-separated email addresses |
| notes | Long text | Documentation/context |

**Error Deduplication Table:**
| Column | Type | Purpose |
|--------|------|---------|
| dedup_key | Formula (Primary) | `{error_type} & "_" & {project_id}` |
| error_type | Single line text | Type of error (e.g., slack_api_error) |
| project_id | Single line text | Project that triggered the error |
| last_sent | Date with time | Timestamp of last email sent |
| count | Number | Count of deduplicated errors |

### Email Template Structure

**Notification Fallback Email:**
```
Subject: [{project_name}] {status} - Slack Notification Failed

Body:
- Header with status badge (Started/Success/Failed)
- Original notification content section
- Error details (workflow, error message, timestamp)
- Footer with n8n admin link
```

**Admin Alert Email:**
```
Subject: [n8n Alert] Slack Integration Failure - {workflow_name}

Body:
- Error summary section
- Affected workflow details
- Remediation steps checklist
- Links: Slack status, n8n admin panel
```

### n8n Credential Names Reference

**Credential names documented for workflow JSON development:**

| Credential | Exact Name | Type |
|------------|------------|------|
| Amazon SES | `Amazon SES - Notifications` | AWS |
| Airtable | `Airtable - n8n Notifications` | Airtable API |

These exact names must be used in workflow JSON files for credential references.
