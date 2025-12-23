# n8n Credentials Setup for Email Fallback

This guide provides step-by-step instructions for configuring the Amazon SES and Airtable credentials in n8n that are required for the email fallback feature.

## Prerequisites

Before proceeding, ensure you have completed:
- [x] Airtable base setup (see [AIRTABLE-SETUP.md](AIRTABLE-SETUP.md))
- [x] Access to your n8n instance at https://nca-n8n.callteksupport.net
- [x] AWS account with SES access
- [x] Airtable personal access token generated

---

## Overview

Two credentials are required for the email fallback feature:

| Credential | Purpose | n8n Name |
|------------|---------|----------|
| Amazon SES | Send fallback notification emails and admin alerts | `Amazon SES - Notifications` |
| Airtable | Query project recipients and manage deduplication | `Airtable - n8n Notifications` |

---

## Part 1: Amazon SES Credential Configuration

Amazon Simple Email Service (SES) is used to send email notifications when Slack delivery fails.

### Step 1.1: Verify Sender Email in AWS SES Console

Before n8n can send emails, you must verify the sender email address in AWS SES.

1. Log in to the [AWS Console](https://console.aws.amazon.com)
2. Navigate to **Amazon Simple Email Service (SES)**
3. In the left sidebar, click **Verified identities**
4. Click **Create identity**
5. Configure the identity:

   | Field | Value |
   |-------|-------|
   | Identity type | **Email address** |
   | Email address | `noreply@callteksupport.com` (or your preferred sender) |

6. Click **Create identity**
7. Check your email inbox for the verification email from AWS
8. Click the verification link in the email

**Important Notes:**
- If your AWS account is in SES Sandbox mode, you must also verify recipient email addresses
- For production use, request to move out of Sandbox mode via AWS Support
- Verification must be completed in the same AWS region you'll use for sending

### Step 1.2: Create AWS IAM User for n8n

Create a dedicated IAM user with SES permissions for n8n:

1. In AWS Console, navigate to **IAM**
2. Click **Users** in the left sidebar
3. Click **Create user**
4. Configure user:

   | Field | Value |
   |-------|-------|
   | User name | `n8n-ses-notifications` |

5. Click **Next**
6. Select **Attach policies directly**
7. Search for and select: `AmazonSESFullAccess`

   *Note: For production, consider creating a custom policy with minimum required permissions:*
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "ses:SendEmail",
           "ses:SendRawEmail"
         ],
         "Resource": "*"
       }
     ]
   }
   ```

8. Click **Next**
9. Click **Create user**

### Step 1.3: Generate Access Keys for the IAM User

1. Click on the newly created user (`n8n-ses-notifications`)
2. Go to the **Security credentials** tab
3. Scroll to **Access keys**
4. Click **Create access key**
5. Select **Third-party service** as the use case
6. Acknowledge the recommendation warning
7. Click **Create access key**
8. **Copy and save both values immediately:**
   - Access key ID: `AKIAXXXXXXXXXXXXXXXX`
   - Secret access key: `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

**Critical:** The secret access key is only shown once. Save it securely before closing this page.

### Step 1.4: Configure SES Credential in n8n

1. Open your n8n instance: https://nca-n8n.callteksupport.net
2. Click **Credentials** in the left sidebar
3. Click **Add Credential**
4. Search for and select **AWS**
5. Configure the credential:

   | Field | Value |
   |-------|-------|
   | **Credential Name** | `Amazon SES - Notifications` |
   | **Region** | `us-east-1` (or your preferred SES region) |
   | **Access Key ID** | Your access key from Step 1.3 |
   | **Secret Access Key** | Your secret access key from Step 1.3 |

6. Click **Save**

### Step 1.5: Test SES Credential

Verify the SES credential works by sending a test email:

1. In n8n, create a new workflow (or use a test workflow)
2. Add an **AWS SES** node
3. Configure the node:

   | Field | Value |
   |-------|-------|
   | **Credential** | `Amazon SES - Notifications` |
   | **Resource** | Email |
   | **Operation** | Send |
   | **From Email** | `noreply@callteksupport.com` |
   | **To Email** | `don@callteksupport.com` |
   | **Subject** | `n8n SES Test - Email Fallback Setup` |
   | **Body** | `This is a test email from n8n to verify SES configuration for the email fallback feature.` |

4. Click **Execute Node**
5. Check that:
   - The node executes successfully (green checkmark)
   - The test email arrives in the recipient's inbox

**Troubleshooting SES Issues:**

| Error | Solution |
|-------|----------|
| "Email address is not verified" | Verify the sender email in SES console (Step 1.1) |
| "Message rejected" | Check if you're in SES Sandbox and recipient is not verified |
| "Access Denied" | Verify IAM user has SES permissions |
| "Invalid region" | Ensure region matches where email is verified |

---

## Part 2: Airtable Credential Configuration

The Airtable credential connects n8n to the "n8n Notifications" base for project lookup and deduplication.

### Step 2.1: Gather Airtable Configuration

Ensure you have the following from the Airtable setup (see [AIRTABLE-SETUP.md](AIRTABLE-SETUP.md)):

| Item | Example | Your Value |
|------|---------|------------|
| Base ID | `appXXXXXXXXXXXXXX` | ______________ |
| Personal Access Token | `patXXX...` | ______________ |
| Projects Table Name | `Projects` | Projects |
| Error Dedup Table Name | `Error Deduplication` | Error Deduplication |

### Step 2.2: Configure Airtable Credential in n8n

1. Open your n8n instance: https://nca-n8n.callteksupport.net
2. Click **Credentials** in the left sidebar
3. Click **Add Credential**
4. Search for and select **Airtable API**
5. Configure the credential:

   | Field | Value |
   |-------|-------|
   | **Credential Name** | `Airtable - n8n Notifications` |
   | **API Key** | Your personal access token from [AIRTABLE-SETUP.md](AIRTABLE-SETUP.md) Step 5 |

6. Click **Save**

**Note:** n8n's Airtable integration uses the Personal Access Token (PAT) in the API Key field. This is the newer authentication method that replaced the legacy API keys.

### Step 2.3: Test Airtable Credential

Verify the Airtable credential works by querying the Projects table:

1. In n8n, create a new workflow (or use a test workflow)
2. Add an **Airtable** node
3. Configure the node:

   | Field | Value |
   |-------|-------|
   | **Credential** | `Airtable - n8n Notifications` |
   | **Resource** | Record |
   | **Operation** | Search |
   | **Base** | n8n Notifications (should auto-populate) |
   | **Table** | Projects |

4. Click **Execute Node**
5. Verify it returns your test project record(s):
   ```json
   {
     "fields": {
       "project_id": "test",
       "project_name": "Test Project",
       "recipients": "don@callteksupport.com",
       "notes": "Initial test project..."
     }
   }
   ```

**Troubleshooting Airtable Issues:**

| Error | Solution |
|-------|----------|
| "Invalid API key" | Regenerate PAT in Airtable with correct scopes |
| "Base not found" | Verify base is included in token's access scope |
| "Table not found" | Check table name spelling (case-sensitive) |
| "INVALID_PERMISSIONS_OR_MODEL_NOT_FOUND" | Token needs `data.records:read` scope |

---

## Credential Reference Summary

After completing setup, your n8n instance should have these credentials configured:

### Credential: Amazon SES - Notifications

| Setting | Value |
|---------|-------|
| **Name** | `Amazon SES - Notifications` |
| **Type** | AWS |
| **Region** | `us-east-1` (or your region) |
| **Verified Sender** | `noreply@callteksupport.com` |
| **Used By** | 04-error-handler.json (SES nodes) |

### Credential: Airtable - n8n Notifications

| Setting | Value |
|---------|-------|
| **Name** | `Airtable - n8n Notifications` |
| **Type** | Airtable API |
| **Token Type** | Personal Access Token |
| **Token Scopes** | `data.records:read`, `data.records:write`, `schema.bases:read` |
| **Base Access** | n8n Notifications (all tables) |
| **Used By** | 04-error-handler.json (Airtable nodes) |

---

## Workflow Integration Notes

When developing the error handler workflow (04-error-handler.json), use these exact credential names:

### SES Node Configuration
```json
{
  "type": "n8n-nodes-base.awsSes",
  "credentials": {
    "aws": {
      "name": "Amazon SES - Notifications"
    }
  }
}
```

### Airtable Node Configuration
```json
{
  "type": "n8n-nodes-base.airtable",
  "credentials": {
    "airtableApi": {
      "name": "Airtable - n8n Notifications"
    }
  }
}
```

---

## Verification Checklist

Before proceeding to workflow development, verify:

### Amazon SES
- [ ] Sender email verified in AWS SES console
- [ ] IAM user created with SES permissions
- [ ] Access key and secret key generated and saved
- [ ] n8n credential `Amazon SES - Notifications` created
- [ ] Test email sent successfully from n8n

### Airtable
- [ ] Personal access token available from Airtable setup
- [ ] n8n credential `Airtable - n8n Notifications` created
- [ ] Test query to Projects table successful
- [ ] Base and tables accessible from n8n

### Both Credentials
- [ ] Credential names match exactly as documented above
- [ ] Credentials saved and visible in n8n Credentials panel

---

## Security Considerations

### AWS Credentials
- Use a dedicated IAM user for n8n (not your personal account)
- Apply principle of least privilege (SES send permissions only)
- Rotate access keys periodically
- Never commit access keys to version control

### Airtable Token
- Use Personal Access Token, not deprecated API key
- Scope token to only the required base
- Limit scopes to minimum needed (read/write records)
- Regenerate token if compromised

### n8n Credential Storage
- n8n encrypts credentials at rest
- Credentials are not exported in workflow JSON by default
- After importing workflows, manually re-link credentials

---

## Next Steps

After completing credential setup:

1. **Create Error Handler Workflow** (Task Group 5)
   - Import or create 04-error-handler.json
   - Connect SES and Airtable credentials to nodes

2. **Add Retry Logic to Existing Workflows** (Task Groups 3-4)
   - Modify 02-deploy-notifications.json
   - Modify 03-slack-interactions.json

3. **Configure Error Workflow Settings** (Task Group 6)
   - Set error workflow on all three existing workflows
   - Test end-to-end with simulated Slack failure

---

## Appendix: AWS SES Region Reference

Choose the SES region closest to your n8n instance for best performance:

| Region Code | Region Name | Notes |
|-------------|-------------|-------|
| `us-east-1` | US East (N. Virginia) | Recommended for US-based services |
| `us-west-2` | US West (Oregon) | Alternative US region |
| `eu-west-1` | Europe (Ireland) | For EU compliance |
| `eu-central-1` | Europe (Frankfurt) | For EU compliance |
| `ap-southeast-1` | Asia Pacific (Singapore) | For APAC services |

Ensure your verified email identity exists in the selected region.
