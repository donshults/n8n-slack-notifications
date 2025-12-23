# Airtable Setup for n8n Notifications

This guide walks through creating the Airtable base that stores project configuration and error deduplication state for the email fallback feature.

## Overview

The Airtable base provides:
1. **Projects Table** - Maps project IDs to email recipients for fallback notifications
2. **Error Deduplication Table** - Tracks sent emails to prevent flooding during outages

---

## Step 1: Create Airtable Base

1. Go to [airtable.com](https://airtable.com) and log in
2. Click **Add a base** (or the + button in your workspace)
3. Select **Start from scratch**
4. Name the base: `n8n Notifications`
5. Click into the base to open it

**Important:** Note the Base ID from the URL. After opening the base, the URL looks like:
```
https://airtable.com/appXXXXXXXXXXXXXX/...
```
The `appXXXXXXXXXXXXXX` part is your **Base ID**. Save this for n8n configuration.

---

## Step 2: Create Projects Table

The default "Table 1" will become your Projects table.

### Rename the Table

1. Double-click on "Table 1" in the tab bar
2. Rename to: `Projects`

### Configure Columns

Delete the default columns and create the following schema:

| Column Name | Field Type | Configuration |
|-------------|------------|---------------|
| `project_id` | Single line text | **Primary field** - unique identifier for each project |
| `project_name` | Single line text | Human-readable project name |
| `recipients` | Long text | Comma-separated email addresses for notifications |
| `notes` | Long text | Optional documentation/context |

### Step-by-Step Column Creation

1. **project_id (Primary Field)**
   - The first column is already the primary field
   - Click the column header dropdown -> **Customize field type**
   - Select **Single line text**
   - Rename to `project_id`

2. **project_name**
   - Click the **+** button to add a column
   - Select **Single line text**
   - Name it `project_name`

3. **recipients**
   - Click **+** to add a column
   - Select **Long text**
   - Name it `recipients`
   - This field holds comma-separated email addresses

4. **notes**
   - Click **+** to add a column
   - Select **Long text**
   - Name it `notes`

5. **Remove unused columns**
   - Delete any auto-generated columns (Notes, Attachments, Status, etc.)
   - Click column header -> **Delete field**

### Final Projects Table Schema

Your table should look like this:

| project_id | project_name | recipients | notes |
|------------|--------------|------------|-------|
| (empty) | (empty) | (empty) | (empty) |

---

## Step 3: Create Error Deduplication Table

1. Click the **+** button next to the "Projects" tab to add a new table
2. Select **Start from scratch**
3. Name the table: `Error Deduplication`

### Configure Columns

| Column Name | Field Type | Configuration |
|-------------|------------|---------------|
| `dedup_key` | Formula | Primary field: `{error_type} & "_" & {project_id}` |
| `error_type` | Single line text | Type of error (e.g., slack_api_error, token_expired) |
| `project_id` | Single line text | Project that triggered the error |
| `last_sent` | Date | Include time field enabled |
| `count` | Number | Integer, default 0 |

### Step-by-Step Column Creation

1. **error_type**
   - The first column starts as primary, but we'll convert it
   - Click the column header -> **Customize field type**
   - Select **Single line text**
   - Name it `error_type`

2. **project_id**
   - Click **+** to add a column
   - Select **Single line text**
   - Name it `project_id`

3. **dedup_key (Formula - Make Primary)**
   - Click **+** to add a column
   - Select **Formula**
   - Name it `dedup_key`
   - Enter the formula:
     ```
     {error_type} & "_" & {project_id}
     ```
   - This creates a unique key like `slack_api_error_context-vault`
   - **Make this the primary field:**
     - Click the `dedup_key` column header
     - Select **Use as primary field** (or drag it to the first position)

4. **last_sent**
   - Click **+** to add a column
   - Select **Date**
   - Name it `last_sent`
   - Enable **Include a time field** option
   - Set time format to your preference (24-hour recommended)

5. **count**
   - Click **+** to add a column
   - Select **Number**
   - Name it `count`
   - Set to **Integer** (no decimal places)

6. **Remove unused columns**
   - Delete any auto-generated columns

### Final Error Deduplication Table Schema

Your table should look like this:

| dedup_key | error_type | project_id | last_sent | count |
|-----------|------------|------------|-----------|-------|
| (formula) | (empty) | (empty) | (empty) | (empty) |

---

## Step 4: Populate Initial Project Configuration

Add at least one test project entry:

### Required: Test Project Entry

In the **Projects** table, add a new row:

| project_id | project_name | recipients | notes |
|------------|--------------|------------|-------|
| `test` | Test Project | `don@callteksupport.com` | Initial test project for email fallback verification |

### Additional Projects

Add entries for any production projects that should receive fallback notifications:

| project_id | project_name | recipients | notes |
|------------|--------------|------------|-------|
| `context-vault` | Context Vault | `don@callteksupport.com` | Main project |
| `example-app` | Example Application | `team@example.com, backup@example.com` | Multiple recipients example |

**Notes:**
- `project_id` must match the `project.id` value sent in webhook payloads
- Multiple recipients should be comma-separated (with or without spaces)
- Projects not in this table will fall back to the default recipient (don@callteksupport.com)

---

## Step 5: Create Airtable API Token

n8n needs API access to query and update the Airtable base.

### Generate Personal Access Token

1. Go to [airtable.com/create/tokens](https://airtable.com/create/tokens)
2. Click **Create new token**
3. Configure the token:

   **Token Name:** `n8n Notifications Integration`

   **Scopes** (add these):
   | Scope | Purpose |
   |-------|---------|
   | `data.records:read` | Read project configuration and dedup state |
   | `data.records:write` | Update deduplication records |
   | `schema.bases:read` | Access base structure (optional but helpful) |

   **Access:**
   - Click **Add a base**
   - Select the `n8n Notifications` base
   - Access level: **All current and future tables in the base**

4. Click **Create token**
5. **Copy the token immediately** - it won't be shown again

### Token Format

The token will look like:
```
patXXXXXXXXXXXXXX.XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

### Secure Token Storage

Store the token securely:
- **Do not commit to git**
- **Do not share in plain text**
- Add to n8n credentials (see Step 6)
- Optionally store in a password manager

---

## Step 6: Configure n8n Airtable Credential

1. Open your n8n instance
2. Go to **Credentials** in the left sidebar
3. Click **Add Credential**
4. Search for and select **Airtable API**
5. Configure:

   | Field | Value |
   |-------|-------|
   | **Credential Name** | `Airtable - n8n Notifications` |
   | **API Key** | Paste the personal access token from Step 5 |

6. Click **Save**

### Test the Credential

To verify the credential works:

1. Create a new workflow
2. Add an **Airtable** node
3. Select your credential: `Airtable - n8n Notifications`
4. Configure:
   - **Resource:** Record
   - **Operation:** Search
   - **Base:** n8n Notifications (should auto-populate)
   - **Table:** Projects
5. Execute the node
6. Verify it returns the test project record

---

## Configuration Summary

After completing setup, you should have:

| Item | Value | Location |
|------|-------|----------|
| Base ID | `appXXXXXXXXXXXXXX` | URL when viewing base |
| Base Name | `n8n Notifications` | Airtable |
| Projects Table ID | `tblXXXXXXXXXXXXXX` | URL when viewing table |
| Error Dedup Table ID | `tblYYYYYYYYYYYYYY` | URL when viewing table |
| API Token | `patXXX...` | Stored in n8n credentials |
| n8n Credential | `Airtable - n8n Notifications` | n8n credentials panel |

**Table IDs:** You can find table IDs in the URL when viewing each table:
```
https://airtable.com/appXXX/tblYYY/...
```

---

## Verification Checklist

Before proceeding to workflow configuration, verify:

- [ ] Base "n8n Notifications" exists
- [ ] Projects table has columns: project_id, project_name, recipients, notes
- [ ] project_id is the primary field in Projects table
- [ ] Error Deduplication table has columns: dedup_key, error_type, project_id, last_sent, count
- [ ] dedup_key formula is `{error_type} & "_" & {project_id}` and is the primary field
- [ ] At least one test project entry exists with `don@callteksupport.com` as recipient
- [ ] Personal access token generated with read/write scopes
- [ ] n8n Airtable credential configured and tested

---

## Troubleshooting

### "Base not found" Error in n8n

- Verify the base is shared with the token (check token access settings)
- Ensure you're using a Personal Access Token, not the deprecated API key
- Check that the token has `data.records:read` scope

### "Table not found" Error in n8n

- Table names are case-sensitive: use `Projects` and `Error Deduplication` exactly
- Verify the table exists in the correct base

### Formula Field Not Calculating

- Ensure both `error_type` and `project_id` have values
- Formula result will be empty if either field is empty
- Check formula syntax: `{error_type} & "_" & {project_id}`

### Token Permission Errors

- Regenerate token with correct scopes
- Ensure "All current and future tables" is selected for base access
- Tokens cannot be edited after creation - create a new one if needed

---

## Next Steps

After Airtable setup is complete:

1. **Configure n8n credentials** - See [N8N-CREDENTIALS-SETUP.md](N8N-CREDENTIALS-SETUP.md) for:
   - Amazon SES credential configuration (for sending fallback emails)
   - Airtable credential verification and testing
2. **Import the error handler workflow** (04-error-handler.json)
3. **Configure error workflow on existing workflows** in n8n UI
4. **Test email fallback** with a simulated Slack failure

---

## Schema Reference

### Projects Table ERD

```
Projects
--------
project_id     : TEXT [PK]
project_name   : TEXT
recipients     : TEXT (comma-separated)
notes          : TEXT
```

### Error Deduplication Table ERD

```
Error_Deduplication
-------------------
dedup_key      : FORMULA [PK] = {error_type} & "_" & {project_id}
error_type     : TEXT
project_id     : TEXT
last_sent      : DATETIME
count          : INTEGER
```

### Deduplication Logic

When an error occurs:
1. Calculate `dedup_key` = `{error_type}_{project_id}`
2. Query Error Deduplication table for matching record
3. If found and `last_sent` is within 5 minutes:
   - Skip sending email
   - Increment `count`
4. If not found or `last_sent` is older than 5 minutes:
   - Send email
   - Create/update record with current timestamp
   - Reset `count` to 1
