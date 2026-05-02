# IMPORT.md - Setup & Deployment Guide

Complete guide to import, configure, and deploy the Automated Client Onboarding workflow in n8n.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Quick Start (5 minutes)](#quick-start-5-minutes)
3. [Detailed Setup](#detailed-setup)
4. [Configuration](#configuration)
5. [Testing](#testing)
6. [Troubleshooting](#troubleshooting)
7. [Deployment](#deployment)
8. [Advanced Configuration](#advanced-configuration)

---

## Prerequisites

Before importing the workflow, you'll need:

### Required
- **n8n instance** (self-hosted or n8n cloud)
  - Version 1.0 or higher
  - Admin access to create credentials
- **Database** (PostgreSQL, MySQL, or SQLite)
  - Create a `users` table with the required schema (see [Database Schema](#database-schema))
  - Connection credentials (host, port, username, password)
- **Email Service** (SendGrid, Postmark, Gmail, or other SMTP)
  - API key or credentials for sending emails

### Optional (For Full Features)
- **Slack workspace** (for notifications)
  - Webhook URL for posting to a channel
- **Cloud storage** (AWS S3, Google Cloud Storage, etc.)
  - For receiving file uploads via URL

### Estimated Time
- Quick setup: 5-10 minutes
- Full setup: 20-30 minutes

---

## Quick Start (5 minutes)

### For Testing Only (Skip This If You Need Real Database)

1. **Import the workflow** (see [Step 1: Import](#step-1-import-the-workflow) below)
2. **Don't configure database** yet - leave as is
3. **Send test data** via webhook
4. **Check execution logs** to verify it parses correctly
5. **Then come back and configure properly**

### For Production

Follow [Detailed Setup](#detailed-setup) below.

---

## Detailed Setup

### Step 1: Import the Workflow

#### Option A: Import from File (Recommended)

1. Open your n8n instance
2. Click **"New Workflow"** (or go to Workflows → Create)
3. Click the **"..."** menu (three dots) in the top right
4. Select **"Import from file"**
5. Select `Automated_Client_Onboarding.json` from this repository
6. Click **"Import"**
7. The workflow canvas will populate with all nodes

#### Option B: Import from URL

If your repository is public, you can import directly:

1. Open n8n
2. Click **"New Workflow"**
3. Click **"..."** → **"Import from URL"**
4. Paste:
   ```
   https://raw.githubusercontent.com/your-username/client-onboarding-automation/main/Automated_Client_Onboarding.json
   ```
5. Click **"Import"**

### Step 2: Save the Workflow

1. Click **"Save"** (Ctrl+S)
2. Name it: `Automated Client Onboarding` (or your preferred name)
3. Click **"Save Workflow"**

Now you have the workflow loaded, but it won't run yet - you need to configure credentials.

---

## Configuration

### Database Setup

The workflow needs database access to check for duplicates and create users.

#### Step 1: Create Database Table

Create a `users` table in your database:

**PostgreSQL:**
```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  full_name VARCHAR(255) NOT NULL,
  role VARCHAR(100) NOT NULL,
  department VARCHAR(100),
  phone VARCHAR(20),
  start_date DATE,
  password_hash VARCHAR(255),
  password_reset_required BOOLEAN DEFAULT true,
  status VARCHAR(50) DEFAULT 'active',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  onboarded_by VARCHAR(255) DEFAULT 'system'
);

-- Create index for faster duplicate detection
CREATE INDEX idx_email ON users(LOWER(email));
```

**MySQL:**
```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  full_name VARCHAR(255) NOT NULL,
  role VARCHAR(100) NOT NULL,
  department VARCHAR(100),
  phone VARCHAR(20),
  start_date DATE,
  password_hash VARCHAR(255),
  password_reset_required BOOLEAN DEFAULT true,
  status VARCHAR(50) DEFAULT 'active',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  onboarded_by VARCHAR(255) DEFAULT 'system'
);

CREATE INDEX idx_email ON users(LOWER(email));
```

**SQLite:**
```sql
CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT UNIQUE NOT NULL,
  full_name TEXT NOT NULL,
  role TEXT NOT NULL,
  department TEXT,
  phone TEXT,
  start_date DATE,
  password_hash TEXT,
  password_reset_required BOOLEAN DEFAULT 1,
  status TEXT DEFAULT 'active',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  onboarded_by TEXT DEFAULT 'system'
);

CREATE INDEX idx_email ON users(LOWER(email));
```

#### Step 2: Add Database Credential to n8n

1. Open your workflow in n8n
2. Find the **"Check Duplicates"** node (around the middle of the workflow)
3. Click on it to open configuration panel
4. Look for **"Postgres"** / **"MySQL"** / **"SQLite"** credential dropdown
5. Click **"Create new"** → **"Postgres"** (or your database type)
6. Fill in credentials:
   - **Host:** Your database host (e.g., `localhost`, `db.example.com`)
   - **Port:** Database port (default: 5432 for PostgreSQL, 3306 for MySQL)
   - **Database:** Database name (e.g., `onboarding_db`)
   - **Username:** Database user (e.g., `postgres`)
   - **Password:** Database password
   - **SSL:** Enable if required by your database provider

7. Click **"Test Connection"** - you should see ✅
8. Click **"Save"**

#### Step 3: Verify Database in Other Nodes

The workflow has multiple database nodes. Repeat step 2 for:
- **"Check Duplicates"** node
- **"Create User"** node

Make sure they all use the same database credential.

#### Step 4: Test Database Connection

1. Add a simple **"Postgres"** node at the end of the workflow temporarily:
   ```sql
   SELECT COUNT(*) as user_count FROM users;
   ```
2. Click **"Execute Node"** (play button)
3. Check the output - should show user count
4. Delete this test node after confirming

---

### Email Setup

The workflow sends welcome emails to new users and admin summaries.

#### Option A: SendGrid (Recommended)

1. Get your SendGrid API key from [sendgrid.com](https://sendgrid.com)
   - Sign up if you don't have an account
   - Go to Settings → API Keys → Create API Key
   - Copy the API key

2. In your n8n workflow, find the **"Send Email"** nodes (there are 2: welcome email + admin report)

3. First email node configuration:
   - Click the node
   - Look for **"Authentication"** → Select **"API Key"**
   - **API Key:** Paste your SendGrid API key
   - **From Email:** `onboarding@yourcompany.com` (must be verified in SendGrid)
   - **To Email:** `{{ $node["Loop"].json.email }}`
   - **Subject:** `Welcome to Company! Here's How to Get Started`
   - **Text/HTML:** Use the template provided

4. Second email node (admin report):
   - Same setup but:
   - **To Email:** `admin@yourcompany.com`
   - **Subject:** `Onboarding Import Complete`
   - **Text/HTML:** Admin report template

#### Option B: Gmail SMTP

1. Enable 2-factor authentication on your Gmail account
2. Create an App Password: [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
3. Copy the 16-character password

4. In n8n, in the **"Send Email"** node:
   - **Authentication:** Select **"SMTP"**
   - **Host:** `smtp.gmail.com`
   - **Port:** `587`
   - **Username:** Your Gmail address
   - **Password:** The 16-character app password
   - **Use TLS:** Toggle ON

5. **From Email:** Your Gmail address
6. Rest of configuration same as SendGrid

#### Option C: Custom SMTP

For Postmark, AWS SES, Office 365, or other providers:

1. Get your SMTP credentials from your email provider
2. In n8n, **"Send Email"** node:
   - **Authentication:** **"SMTP"**
   - **Host:** Your provider's SMTP host
   - **Port:** Usually 587 (TLS) or 465 (SSL)
   - **Username:** Usually your email
   - **Password:** Your password or API key
   - **Use TLS/SSL:** Check your provider's requirements

---

### Slack Notifications (Optional)

To get instant notifications when onboarding completes:

1. Create a Slack App or Webhook:
   - Go to [api.slack.com/apps](https://api.slack.com/apps)
   - Click **"Create New App"** → **"From scratch"**
   - Name: `Onboarding Notifications`
   - Workspace: Your workspace
   - Click **"Create App"**

2. Enable Incoming Webhooks:
   - In app settings, click **"Incoming Webhooks"**
   - Toggle **"Activate Incoming Webhooks"** ON
   - Click **"Add New Webhook to Workspace"**
   - Select channel where notifications go (e.g., `#onboarding`)
   - Click **"Allow"**
   - Copy the Webhook URL

3. In your n8n workflow, find the **"Slack"** node (if you added one):
   - **Webhook URL:** Paste your Slack webhook URL
   - **Channel:** `#onboarding-notifications`
   - **Message:** Use the template provided

---

## Testing

### Test 1: Verify Import Works

1. Open the workflow canvas
2. You should see nodes for:
   - Webhook (input)
   - Parse/Extract (data processing)
   - Loop (iteration)
   - Validation nodes
   - Database nodes
   - Email nodes
3. No red error indicators should be visible

### Test 2: Send Sample Data via Webhook

#### Get Your Webhook URL

1. Click the **"Webhook"** node
2. Copy the **Webhook URL** from the panel

#### Send Test Data

**Using curl (macOS/Linux):**
```bash
curl -X POST https://your-webhook-url \
  -H "Content-Type: application/json" \
  -d '{
    "client_id": "test_client_1",
    "client_name": "Test Company",
    "data": [
      {
        "email": "john.smith@test.com",
        "full_name": "John Smith",
        "role": "Manager",
        "department": "Sales",
        "phone": "555-1234",
        "start_date": "2024-01-15"
      }
    ]
  }'
```

**Using Postman:**
1. Open Postman
2. Create new request
3. Method: **POST**
4. URL: Paste your webhook URL
5. Body tab → Select **"raw"** → **"JSON"**
6. Paste the test data above
7. Click **"Send"**

#### Check Results

1. Look at execution logs in n8n
2. Check your database:
   ```sql
   SELECT * FROM users ORDER BY created_at DESC LIMIT 1;
   ```
3. Check your inbox for welcome email
4. Look for admin email to `admin@yourcompany.com`

### Test 3: Upload Sample Data File

To test file upload via URL:

1. Host `sample-data.csv` somewhere accessible (e.g., GitHub raw, S3, etc.)
2. Get the URL: `https://example.com/sample-data.csv`
3. Send via webhook:
   ```bash
   curl -X POST https://your-webhook-url \
     -H "Content-Type: application/json" \
     -d '{
       "client_id": "test_client_2",
       "client_name": "Acme Corp",
       "file_url": "https://example.com/sample-data.csv",
       "file_format": "csv"
     }'
   ```

### Test 4: Full End-to-End Test

1. Use `sample-data.csv` (13 employees)
2. Send via file URL
3. Watch execution
4. Verify:
   - 13 users created in database (or 12 if 1 duplicate)
   - 13 welcome emails sent
   - 1 admin report email
   - Execution completed without errors

---

## Troubleshooting

### Issue: Webhook Not Receiving Data

**Symptom:** No execution logs when sending data

**Solutions:**
1. Verify webhook URL is copied correctly
2. Check URL includes `https://`
3. Make sure workflow is **saved** and **activated**
4. Check firewall isn't blocking the request
5. Try from different network
6. Check request headers include `Content-Type: application/json`

---

### Issue: "Database Connection Failed"

**Symptom:** Red error on database nodes

**Solutions:**
1. Verify database is running
2. Test credentials manually:
   ```bash
   psql -h your-host -U your-user -d your-database
   ```
3. Check firewall allows connection to database port
4. Verify user has permission to read/write `users` table
5. In n8n, click "Test Connection" on credential
6. Check if database requires SSL/TLS

---

### Issue: "Invalid Email Format"

**Symptom:** Validation error on email field

**Solutions:**
1. Ensure email field contains valid email address
2. Check for whitespace before/after email
3. Verify email field name matches exactly: `email` or `Email` (depending on CSV)
4. Look at the error message for exact issue

---

### Issue: "Email Not Sending"

**Symptom:** Users created but no email received

**Solutions:**
1. Check SendGrid API key is correct
2. Verify "From Email" is authorized in SendGrid
3. Look at execution logs for error message
4. Check email template doesn't have broken variables
5. Verify recipient email is valid
6. Check spam/junk folder
7. Try sending test email from SendGrid dashboard first

---

### Issue: "Duplicate Email Not Detected"

**Symptom:** User created twice with same email

**Solutions:**
1. Verify email is actually lowercase in database
2. Check database index was created:
   ```sql
   SELECT * FROM information_schema.statistics WHERE table_name='users' AND column_name='email';
   ```
3. Try clearing database and reimporting
4. Check table has `UNIQUE` constraint on email column

---

### Issue: "CSV Not Parsing"

**Symptom:** Error in parse node, invalid CSV

**Solutions:**
1. Verify CSV is valid (open in Excel/Google Sheets)
2. Check column names match exactly (case-sensitive)
3. Ensure no special characters in headers
4. Check file encoding is UTF-8 (not UTF-16 or Latin-1)
5. Verify CSV is accessible at file_url
6. Try with sample-data.csv from this repo first

---

### Issue: "Workflow Execution Timeout"

**Symptom:** Workflow takes >30 minutes

**Solutions:**
1. Reduce batch size (process fewer records per execution)
2. Check database performance (slow inserts)
3. Look for loops that never terminate
4. Check if emails are taking too long to send
5. In n8n Cloud, increase timeout in workflow settings

---

### Issue: "Out of Memory"

**Symptom:** n8n crashes or becomes unresponsive

**Solutions:**
1. Reduce number of records processed per batch (e.g., process 1,000 at a time instead of 10,000)
2. Clear execution history in n8n
3. Restart n8n instance
4. Check for memory leaks in custom code nodes
5. For large batches, use separate runs instead of one big batch

---

### Debug Mode

To enable detailed logging:

1. In n8n, click **Workflow → Settings**
2. Enable **"Save execution progress"**
3. Enable **"Save manual executions"**
4. Re-run workflow and check execution logs
5. Look for detailed error messages and variables

---

## Deployment

### Development vs. Production

#### Development (Testing Only)
- Run on localhost
- Use test database
- Send emails to test address
- No activation needed

#### Production (Live)
- Run on cloud instance or server
- Use production database
- Send emails to real addresses
- Activate workflow

---

### Activating the Workflow

Once testing is complete:

1. Click **"Activate"** toggle (top left, currently OFF)
2. Toggle turns blue → Workflow is now active
3. Webhook will now accept requests
4. Set up any monitoring/alerting you need

---

### Scheduling (Optional)

To run onboarding on a schedule instead of via webhook:

1. Remove or disable **Webhook** trigger
2. Add **Schedule** node
3. Configure frequency (daily, weekly, etc.)
4. Connect to data source (load CSV from cloud storage)
5. Activate

---

### Monitoring

#### Check Recent Executions

1. Click **"Executions"** tab
2. See all runs with:
   - Execution time
   - Status (success/error)
   - Data processed
3. Click each execution to see details

#### Set Up Alerts (Optional)

1. Go to **"Settings"** → **"Notifications"**
2. Configure email alerts for:
   - Workflow failures
   - Long execution times
   - High error rates

#### Check Logs

```bash
# If self-hosted n8n:
tail -f ~/.n8n/log.txt
```

---

## Advanced Configuration

### Custom Email Templates

To customize the welcome email:

1. Find the **"Send Email"** node (first one, for users)
2. In the HTML/Text field, modify the template:

```html
Hi {{ $node["Normalize_Name"].json.full_name }},

Your account has been created and is ready to use.

📧 Email: {{ $node["Normalize_Email"].json.email }}
🔑 Temporary Password: {{ $node["Generate_Password"].json.temp_password }}
🔗 Login: https://app.yourcompany.com/login

Next Steps:
1. Log in with your temporary password
2. Change it to something secure
3. Review your onboarding checklist
4. Join our Slack workspace

Welcome to the team!

Best regards,
The Onboarding Team
```

### Custom Validation Rules

To add additional validation (e.g., require specific domain):

1. Find **"Validate Email"** node
2. Modify the regex:

```javascript
// Require @company.com domain
const emailRegex = /^[^\s@]+@company\.com$/;
if (!emailRegex.test(email)) {
  errors.push('Email must be @company.com');
}
```

### Modify Database Schema

If your table has different columns:

1. Find **"Create User"** node
2. Modify the INSERT query to match your columns:

```sql
INSERT INTO my_users_table (
  email,
  full_name,
  role,
  department,
  phone,
  start_date,
  custom_field_1,
  custom_field_2
) VALUES (?, ?, ?, ?, ?, ?, ?, ?)
```

3. Add parameters for new fields

### Add Webhook Authentication

To protect your webhook from unauthorized requests:

1. Click **Webhook** node
2. Expand **"Authentication"**
3. Select **"Basic Auth"** or **"Header Auth"**
4. Set username/password or header key
5. Clients must provide credentials when calling webhook

---

## Getting Help

### Resources

- **n8n Docs:** [docs.n8n.io](https://docs.n8n.io)
- **n8n Community:** [community.n8n.io](https://community.n8n.io)
- **GitHub Issues:** Open an issue in this repository
- **Email:** See repository contact info

### Providing Debug Info

If you need help, provide:

1. Workflow export (Settings → Download)
2. Execution logs (from failed execution)
3. Database schema (`SHOW CREATE TABLE users;`)
4. Error message (full text)
5. Steps to reproduce

---

## Checklist Before Going Live

- [ ] Database table created and tested
- [ ] Database credentials added to n8n
- [ ] Email service configured and tested
- [ ] Sample data imported successfully
- [ ] 13 test users created in database
- [ ] Welcome emails received
- [ ] Admin emails received
- [ ] Duplicate detection working
- [ ] Validation catching bad data
- [ ] Workflow saved
- [ ] Webhook URL copied and tested
- [ ] Slack notifications set up (optional)
- [ ] Team trained on how to use workflow
- [ ] Backup of database before going live
- [ ] Monitoring/alerts configured

---

## Rollback Plan

If something goes wrong in production:

1. Click **"Deactivate"** to stop accepting new requests
2. Check execution logs for errors
3. Fix the issue (database, email, validation, etc.)
4. Test with sample data
5. Re-activate when ready

If data was corrupted:

1. Restore database from backup
2. Re-run onboarding for affected clients
3. Send email to admins explaining what happened

---

## Next Steps

1. Import the workflow (follow [Step 1](#step-1-import-the-workflow) above)
2. Configure database (follow [Database Setup](#database-setup))
3. Set up email (follow [Email Setup](#email-setup))
4. Test with sample data (follow [Testing](#testing))
5. Activate and deploy

You now have a production-ready client onboarding automation!

---

## Support

If you hit issues:

1. Check [Troubleshooting](#troubleshooting) section
2. Review execution logs in n8n
3. Test individual nodes
4. Check database connectivity
5. Verify credentials are correct
6. Open GitHub issue with debug info

Happy onboarding! 🚀
