# Workflow Architecture

A detailed walkthrough of how the automated client onboarding workflow operates. This is the technical deep dive—what happens at each step, why decisions are made, and how to modify it.

## System Overview

The workflow processes client employee data from upload to fully onboarded users. It's designed to handle messy input, recover from errors, and provide complete visibility into what happened.

```
Trigger
  ↓
Parse Input (CSV/JSON)
  ↓
Loop Through Rows
  ├─ Validate
  ├─ Transform
  ├─ Check Duplicates
  └─ Create User
  ↓
Send Communications
  ├─ Welcome Emails
  ├─ Admin Report
  └─ Slack Notification
  ↓
Log Metrics
```

## Detailed Step-by-Step

### 1. Trigger: Receive Data

**Node Type:** Webhook

**What it does:**
Listen for incoming requests containing file URLs or data directly.

**Expected input:**
```json
{
  "file_url": "https://storage.example.com/acme_employees.csv",
  "client_id": "client_789",
  "client_name": "Acme Corp",
  "file_format": "csv"
}
```

**Or:**
```json
{
  "data": [
    {
      "email": "john@acme.com",
      "full_name": "John Smith",
      "role": "Manager",
      "department": "Sales",
      "phone": "555-1234",
      "start_date": "01/15/2024"
    }
  ],
  "client_id": "client_789",
  "client_name": "Acme Corp"
}
```

**Why this design:**
- Webhook allows real-time triggering from your app/form
- Accepts either file URL (for large imports) or direct data (for real-time integrations)
- Client metadata passed in so reports know who this is for

---

### 2. Download & Parse File

**Node Type:** HTTP Request + Code Node

**If file_url is provided:**

```javascript
// HTTP Request node
GET https://storage.example.com/acme_employees.csv

// Returns file content as string
```

**Parse based on format:**

```javascript
// Code node: parseFile()

const fileContent = $input.body;
const fileFormat = $input.json.file_format;

if (fileFormat === 'csv') {
  // Use CSV parser (Papa Parse in n8n)
  const rows = parseCSV(fileContent);
  return { data: rows };
}

if (fileFormat === 'json') {
  // Parse JSON directly
  const data = JSON.parse(fileContent);
  return { data: data.employees || data };
}

if (fileFormat === 'xlsx') {
  // Use Excel parser
  const rows = parseExcel(fileContent);
  return { data: rows };
}

// If format unknown, try to detect
throw new Error(`Unsupported format: ${fileFormat}`);
```

**Output:**
Array of objects, one per row:
```json
[
  {
    "email": "john@acme.com",
    "full_name": "John Smith",
    "role": "Manager",
    "department": "Sales",
    "phone": "555-1234",
    "start_date": "01/15/2024"
  },
  {
    "email": "jane@acme.com",
    "full_name": "Jane Doe",
    ...
  }
]
```

**Error handling:**
- If file can't be downloaded → catch error, email client, stop
- If format is unknown → try JSON parser as fallback
- If both fail → log and notify admin

---

### 3. Initialize Tracking Arrays

**Node Type:** Set Variable

Before looping, initialize arrays to track success/failure:

```javascript
{
  "successful_users": [],
  "failed_rows": [],
  "existing_users": [],
  "total_processed": 0,
  "start_time": new Date().toISOString()
}
```

This is where we'll store results as we loop through each row.

---

### 4. Loop Through Each Row

**Node Type:** Loop / For Each

**Iterates over:** The parsed data array

**For each iteration:**
Process one employee record from the upload

```
Row Input: { email: "john@acme.com", full_name: "John Smith", ... }
            ↓
            Validate
            ↓
            Transform
            ↓
            Check Database
            ↓
            Create User or Skip
            ↓
            Add to success/failure array
```

---

### 5A. Validation: Check Required Fields

**Node Type:** Code Node (inside loop)

**For each row, check:**

```javascript
// validateRow()

const row = $input.json;
const errors = [];

// Email is required and must be valid
if (!row.email || row.email.trim() === '') {
  errors.push('Missing email address');
} else if (!isValidEmail(row.email)) {
  errors.push(`Invalid email format: ${row.email}`);
}

// Full name is required
if (!row.full_name || row.full_name.trim() === '') {
  errors.push('Missing full name');
}

// Role is required and must match enum
if (!row.role || row.role.trim() === '') {
  errors.push('Missing role');
} else {
  const validRoles = ['Manager', 'Engineer', 'Support', 'Sales', 'HR', 'QA', 'Product Manager'];
  if (!validRoles.includes(normalizeRole(row.role))) {
    errors.push(`Unknown role: ${row.role}. Valid roles: ${validRoles.join(', ')}`);
  }
}

// Phone is optional, but if present, should be valid
if (row.phone && row.phone.trim() !== '') {
  if (!isValidPhoneFormat(row.phone)) {
    errors.push(`Invalid phone format: ${row.phone}`);
  }
}

// Start date is required and should be parseable
if (!row.start_date || row.start_date.trim() === '') {
  errors.push('Missing start date');
} else if (!canParseDate(row.start_date)) {
  errors.push(`Cannot parse date: ${row.start_date}`);
}

return {
  row: row,
  is_valid: errors.length === 0,
  errors: errors,
  error_message: errors.join('; ')
};
```

**Helper function: Email validation**

```javascript
function isValidEmail(email) {
  // RFC 5322 simplified regex
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return regex.test(email.trim().toLowerCase());
}
```

**Helper function: Date parsing**

```javascript
function canParseDate(dateString) {
  // Try parsing multiple common formats
  const formats = [
    /^\d{1,2}\/\d{1,2}\/\d{4}$/,      // MM/DD/YYYY
    /^\d{4}-\d{1,2}-\d{1,2}$/,        // YYYY-MM-DD
    /^\d{1,2}-\d{1,2}-\d{4}$/,        // DD-MM-YYYY or MM-DD-YYYY
    /^[A-Za-z]+ \d{1,2}, \d{4}$/,     // Jan 1, 2024
    /^[A-Za-z]+ \d{1,2} \d{4}$/,      // Jan 1 2024
  ];
  
  return formats.some(format => format.test(dateString.trim()));
}
```

**Output:**
```json
{
  "is_valid": true,
  "errors": [],
  "error_message": null,
  "row": { ... }
}
```

---

### 5B. Validation: Check Duplicates (Optional, Early Check)

**Node Type:** Database Query (optional optimization)

Query existing users before transformation:

```sql
SELECT email FROM users 
WHERE email = LOWER(TRIM(?))
```

If email exists → mark as "already_exists" (not an error, just skip)

This early check saves processing time on rows we'll skip anyway.

---

### 6A. Transformation: Normalize Email

**Node Type:** Code Node

```javascript
// normalizeEmail()

const email = row.email.trim().toLowerCase();

return {
  email: email,
  email_original: row.email
};
```

**Examples:**
```
"  John.Smith@Acme.COM  " → "john.smith@acme.com"
"JANE_DOE@COMPANY.COM" → "jane_doe@company.com"
```

---

### 6B. Transformation: Normalize Name

**Node Type:** Code Node

```javascript
// normalizeName()

function toTitleCase(str) {
  return str
    .toLowerCase()
    .split(' ')
    .map(word => word.charAt(0).toUpperCase() + word.slice(1))
    .join(' ');
}

const fullName = toTitleCase(row.full_name.trim());

return {
  full_name: fullName,
  full_name_original: row.full_name
};
```

**Examples:**
```
"john smith" → "John Smith"
"JANE DOE" → "Jane Doe"
"mary-ann jones" → "Mary-ann Jones"
```

---

### 6C. Transformation: Normalize Phone

**Node Type:** Code Node

```javascript
// normalizePhone()

function normalizePhoneToE164(phoneString) {
  if (!phoneString) return null;
  
  // Remove all non-numeric except leading +
  let cleaned = phoneString.trim().replace(/[^\d+]/g, '');
  
  // If no country code, assume US (+1)
  if (!cleaned.startsWith('+')) {
    cleaned = '+1' + cleaned;
  }
  
  // Remove formatting from country code
  cleaned = cleaned.replace(/\D/g, '');
  cleaned = '+' + cleaned;
  
  // Validate length (E.164 is 7-15 digits)
  const digitCount = cleaned.replace(/\D/g, '').length;
  if (digitCount < 7 || digitCount > 15) {
    return { phone: null, error: `Invalid phone length: ${phoneString}` };
  }
  
  return {
    phone: cleaned,
    phone_original: phoneString
  };
}
```

**Examples:**
```
"555-1234" → "+15551234" (incomplete, but keeps as-is)
"(555) 123-4567" → "+15551234567"
"5551234567" → "+15551234567"
"+1-555-123-4567" → "+15551234567"
" 555 444 3333 " → "+15554443333"
```

---

### 6D. Transformation: Normalize Date

**Node Type:** Code Node

```javascript
// normalizeDate()

function parseToISO8601(dateString) {
  if (!dateString) return null;
  
  const str = dateString.trim();
  let date;
  
  // Try MM/DD/YYYY format
  let match = str.match(/^(\d{1,2})\/(\d{1,2})\/(\d{4})$/);
  if (match) {
    date = new Date(match[3], match[1] - 1, match[2]);
  }
  
  // Try YYYY-MM-DD format
  match = str.match(/^(\d{4})-(\d{1,2})-(\d{1,2})$/);
  if (match) {
    date = new Date(match[1], match[2] - 1, match[3]);
  }
  
  // Try DD-MM-YYYY format (guess based on day > 12)
  match = str.match(/^(\d{1,2})-(\d{1,2})-(\d{4})$/);
  if (match) {
    const day = parseInt(match[1]);
    const month = parseInt(match[2]);
    // If day > 12, it's probably DD-MM-YYYY
    if (day > 12) {
      date = new Date(match[3], month - 1, day);
    } else {
      date = new Date(match[3], day - 1, month);
    }
  }
  
  // Try written format: "Jan 15, 2024" or "Jan 15 2024"
  const monthNames = {
    'jan': 0, 'feb': 1, 'mar': 2, 'apr': 3, 'may': 4, 'jun': 5,
    'jul': 6, 'aug': 7, 'sep': 8, 'oct': 9, 'nov': 10, 'dec': 11
  };
  
  match = str.match(/^([A-Za-z]+)\s+(\d{1,2}),?\s+(\d{4})$/);
  if (match) {
    const monthIndex = monthNames[match[1].toLowerCase().slice(0, 3)];
    date = new Date(match[3], monthIndex, match[2]);
  }
  
  // If we got a valid date, convert to ISO 8601
  if (date && date instanceof Date && !isNaN(date)) {
    const iso = date.toISOString().split('T')[0];
    return {
      start_date: iso,
      start_date_original: dateString
    };
  }
  
  return {
    start_date: null,
    error: `Cannot parse date: ${dateString}`
  };
}
```

**Examples:**
```
"01/15/2024" → "2024-01-15"
"Jan 15, 2024" → "2024-01-15"
"2024-01-15" → "2024-01-15"
"15-01-2024" → "2024-01-15"
"3/5/2024" → "2024-03-05"
"2024.02.01" → "2024-02-01"
```

---

### 6E. Transformation: Map Role

**Node Type:** Code Node

```javascript
// normalizeRole()

const roleMapping = {
  // Manager variations
  'manager': 'MANAGER',
  'mgr': 'MANAGER',
  'mngr': 'MANAGER',
  
  // Engineer variations
  'engineer': 'ENGINEER',
  'eng': 'ENGINEER',
  'developer': 'ENGINEER',
  'dev': 'ENGINEER',
  
  // Support variations
  'support': 'SUPPORT',
  'support rep': 'SUPPORT',
  'customer success': 'SUPPORT',
  'cs': 'SUPPORT',
  
  // Sales variations
  'sales': 'SALES',
  'sales rep': 'SALES',
  'account manager': 'SALES',
  'account executive': 'SALES',
  
  // HR variations
  'hr': 'HR',
  'human resources': 'HR',
  'recruiter': 'HR',
  
  // QA variations
  'qa': 'QA',
  'quality assurance': 'QA',
  'tester': 'QA',
  
  // Product
  'product manager': 'PRODUCT_MANAGER',
  'product': 'PRODUCT_MANAGER',
  'pm': 'PRODUCT_MANAGER',
};

const inputRole = row.role.trim().toLowerCase();
const mappedRole = roleMapping[inputRole] || null;

if (!mappedRole) {
  return {
    role: null,
    error: `Unknown role: ${row.role}. Cannot map to system enum.`
  };
}

return {
  role: mappedRole,
  role_original: row.role
};
```

**Examples:**
```
"manager" → "MANAGER"
"MANAGER" → "MANAGER"
"mngr" → "MANAGER"
"engineer" → "ENGINEER"
"eng" → "ENGINEER"
"developer" → "ENGINEER"
```

---

### 6F. Transformation: Assign Department Default

**Node Type:** Code Node

```javascript
// assignDepartmentDefault()

const departmentDefaults = {
  'MANAGER': 'Administration',
  'ENGINEER': 'Engineering',
  'SUPPORT': 'Customer Success',
  'SALES': 'Sales',
  'HR': 'Human Resources',
  'QA': 'Quality Assurance',
  'PRODUCT_MANAGER': 'Product'
};

let department = row.department;

if (!department || department.trim() === '') {
  // Assign default based on role
  department = departmentDefaults[row.role] || 'General';
}

return {
  department: department.trim(),
  department_original: row.department
};
```

**Examples:**
```
If role is "ENGINEER" and department is blank → "Engineering"
If role is "MANAGER" and department is blank → "Administration"
If department is provided → use as-is (after trimming)
```

---

### 7. Check Database for Duplicates

**Node Type:** Database Query (SQL)

Before creating the user, check if email exists:

```sql
SELECT id, email, created_at 
FROM users 
WHERE LOWER(email) = LOWER(?)
LIMIT 1
```

**Parameter:** normalized email from step 6A

**Decision Logic:**

```javascript
// checkDuplicate()

const existingUser = queryResult;

if (existingUser && existingUser.id) {
  return {
    action: 'skip',
    reason: 'already_exists',
    message: `User already exists (ID: ${existingUser.id}, created: ${existingUser.created_at})`
  };
}

return {
  action: 'proceed_with_creation'
};
```

**Output:**
- If duplicate found → skip creation, add to "existing_users" array
- If new → proceed to user creation

---

### 8. Create User in Database

**Node Type:** Database Insert (SQL)

Only if validation passed and user doesn't exist:

```sql
INSERT INTO users (
  email,
  full_name,
  role,
  department,
  phone,
  start_date,
  password_hash,
  password_reset_required,
  status,
  created_at,
  onboarded_by
) VALUES (
  ?,
  ?,
  ?,
  ?,
  ?,
  ?,
  ?,
  true,
  'active',
  NOW(),
  'system'
)
RETURNING id, email, created_at;
```

**Parameters:**
```javascript
// generateTempPassword()
function generateTempPassword() {
  const uppercase = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
  const lowercase = 'abcdefghijklmnopqrstuvwxyz';
  const numbers = '0123456789';
  const symbols = '!@#$%^&*';
  
  let password = '';
  password += uppercase[Math.floor(Math.random() * uppercase.length)];
  password += lowercase[Math.floor(Math.random() * lowercase.length)];
  password += numbers[Math.floor(Math.random() * numbers.length)];
  password += symbols[Math.floor(Math.random() * symbols.length)];
  
  // Add 4 more random characters
  const all = uppercase + lowercase + numbers;
  for (let i = 0; i < 4; i++) {
    password += all[Math.floor(Math.random() * all.length)];
  }
  
  // Shuffle password
  password = password.split('').sort(() => 0.5 - Math.random()).join('');
  
  return password;
}

const tempPassword = generateTempPassword();
const passwordHash = bcrypt.hashSync(tempPassword, 10); // Hash before storing
```

**Output:**
```json
{
  "user_id": "user_789",
  "email": "john@acme.com",
  "full_name": "John Smith",
  "created_at": "2024-04-30T14:30:00Z",
  "temp_password": "aB3!xYzQ" // Store separately (never in logs)
}
```

---

### 9. Conditional: Route Based on Validation

**Node Type:** IF/Switch Node

```javascript
// Route based on validation result
if (validation_result.is_valid && duplicate_check.action === 'proceed_with_creation') {
  // Go to Step 8: Create User
  return 'create_user';
}

if (validation_result.is_valid && duplicate_check.action === 'skip') {
  // Skip, add to existing_users
  return 'already_exists';
}

if (!validation_result.is_valid) {
  // Add to failed_rows
  return 'validation_failed';
}
```

---

### 10. Update Tracking Arrays

**Node Type:** Code Node (after each loop iteration)

Add the result to the appropriate tracking array:

```javascript
// updateTracking()

const tracking = $input.json.tracking; // From step 3

if (loop_result.success) {
  tracking.successful_users.push({
    email: loop_result.email,
    user_id: loop_result.user_id,
    created_at: loop_result.created_at
  });
} else if (loop_result.reason === 'already_exists') {
  tracking.existing_users.push({
    email: loop_result.email,
    reason: loop_result.message
  });
} else {
  tracking.failed_rows.push({
    row_data: loop_result.row,
    errors: loop_result.errors
  });
}

tracking.total_processed += 1;

return tracking;
```

---

### 11. Send Welcome Emails

**Node Type:** Loop through successful_users + Email Node

After all users created, send welcome email to each:

```
To: john@acme.com
From: onboarding@company.com
Subject: Welcome to Company! Here's How to Get Started

Hi John,

Your account has been created and is ready to use.

📧 Email: john@acme.com
🔑 Temporary Password: aB3!xYzQ (change on first login)
🔗 Login: https://app.company.com/login

Next Steps:
1. Log in with your temporary password
2. You'll be prompted to change it to something secure
3. Review your onboarding checklist (attached)
4. Join our Slack workspace: https://company.slack.com/join
5. Meet with your manager on [start_date]

Need help? Reply to this email or contact support@company.com

Welcome aboard!
```

**Attachment:**
- Generated PDF onboarding checklist (see next step)

**Error handling:**
- If email fails to send → log it, retry 3 times, notify admin if all fail

---

### 12. Generate PDF Onboarding Checklist

**Node Type:** Code Node + PDF Generator

Generate personalized PDF for each user:

```javascript
// generateOnboardingChecklist()

const user = successful_users[index];

const checklist = `
ONBOARDING CHECKLIST
${user.full_name} | ${user.role} | ${user.department}
Start Date: ${user.start_date}

WEEK 1 - Getting Started
☐ Complete your profile setup
☐ Change temporary password
☐ Enable two-factor authentication
☐ Review company handbook
☐ Watch product training videos
☐ Join Slack workspace
☐ Add to relevant channels

WEEK 2 - Meet the Team
☐ Attend company orientation (Tuesday 10am)
☐ 1-on-1 with manager (schedule)
☐ Meet your team members
☐ Review role responsibilities
☐ Set up dev environment (if engineer)
☐ Get hardware setup (if needed)

WEEK 3-4 - Ramping Up
☐ Complete assigned training modules
☐ Attend team standups
☐ Review processes and documentation
☐ First project assignment
☐ Coffee chats with cross-functional teams

ADMIN: ${user.full_name}
Created: ${new Date().toISOString().split('T')[0]}
ID: ${user.user_id}
`;

// Convert to PDF
const pdf = generatePDF(checklist);
return { pdf: pdf, user_email: user.email };
```

---

### 13. Generate Admin Report

**Node Type:** Code Node

Create summary report for admin/client:

```javascript
// generateAdminReport()

const tracking = $input.json.tracking;
const client = $input.json.client;
const endTime = new Date();
const duration = (endTime - new Date(tracking.start_time)) / 1000 / 60; // minutes

const report = `
ONBOARDING IMPORT SUMMARY

Client: ${client.client_name}
Timestamp: ${endTime.toISOString()}
Duration: ${duration.toFixed(1)} minutes

RESULTS
✅ Successfully Created: ${tracking.successful_users.length}
⏳ Already Existed: ${tracking.existing_users.length}
❌ Errors: ${tracking.failed_rows.length}
📊 Total Processed: ${tracking.total_processed}

SUCCESS RATE: ${((tracking.successful_users.length / tracking.total_processed) * 100).toFixed(1)}%

SUCCESSFULLY CREATED (${tracking.successful_users.length})
${tracking.successful_users.map(u => `✓ ${u.email} | ID: ${u.user_id}`).join('\n')}

ALREADY EXISTED (${tracking.existing_users.length})
${tracking.existing_users.map(u => `⏳ ${u.email}`).join('\n')}

ERRORS (${tracking.failed_rows.length})
${tracking.failed_rows.map(f => `❌ Row data: ${JSON.stringify(f.row_data)}\n   Errors: ${f.errors.join('; ')}`).join('\n')}

NEXT STEPS
1. New users should change passwords within 48 hours
2. Admins can verify creation in user management
3. All users should complete onboarding checklist
4. Monitor for failed first logins
`;

return { report: report };
```

---

### 14. Send Admin Email

**Node Type:** Email Node

```
To: admin@company.com
From: system@company.com
Subject: Onboarding Import Complete - [Client Name]

[Include report from step 13]

Attachments: detailed_import_log.json
```

---

### 15. Send Slack Notification

**Node Type:** Slack Node

```
Channel: #onboarding-notifications
Icon: 🎉

Onboarding Import Complete
Client: Acme Corp
✅ Created: 13 users
⏳ Existed: 2 users
❌ Errors: 0
⏱️ Time: 8 minutes

<https://admin.company.com/imports/12345|View Full Report>
```

---

### 16. Log Metrics to Analytics

**Node Type:** Database Insert or Analytics Service

```javascript
// logMetrics()

const metrics = {
  event: 'onboarding_import_complete',
  client_id: $input.json.client.client_id,
  client_name: $input.json.client.client_name,
  timestamp: new Date().toISOString(),
  users_created: tracking.successful_users.length,
  users_skipped: tracking.existing_users.length,
  errors: tracking.failed_rows.length,
  total_processed: tracking.total_processed,
  success_rate: (tracking.successful_users.length / tracking.total_processed),
  execution_time_seconds: duration * 60,
  file_format: $input.json.file_format
};

// Insert into analytics table or send to service
INSERT INTO workflow_metrics (...) VALUES (...)
// Or: POST to analytics API
```

---

## Error Handling Strategy

### Scenario 1: File Download Fails
```
→ Catch error
→ Send email to client with error details
→ Stop workflow (don't retry—file issue on their end)
→ Alert admin
```

### Scenario 2: Database Connection Fails
```
→ Pause for 30 seconds
→ Retry up to 3 times with exponential backoff
→ If still failing: queue job for later, don't lose data
→ Send admin alert
```

### Scenario 3: Single Row Validation Fails
```
→ Add to failed_rows array
→ Continue processing other rows
→ Include in admin report
→ Don't block entire import
```

### Scenario 4: Email Send Fails
```
→ Log failure
→ Retry 2 more times
→ If all fail: notify admin, mark user as "needs_email_retry"
→ Continue with next user
```

### Scenario 5: Duplicate Email Found
```
→ Don't create user (would cause database constraint error)
→ Add to "existing_users" (not an error, just skip)
→ Include in report as informational
```

---

## Performance Considerations

**For 100 users:**
- Parse: ~100ms
- Validate: ~500ms (50ms per row)
- Check duplicates: ~2-3s (depends on DB query speed)
- Transform: ~200ms
- Create users: ~3-5s (depends on DB insert speed)
- Send emails: ~30-60s (SendGrid API calls)
- **Total: ~1-2 minutes**

**For 1,000 users:**
- Same process, but consider:
  - Batch database inserts (instead of 1 per row)
  - Batch email sends (queue them, process async)
  - Add progress tracking
  - **Total: ~8-15 minutes**

**For 10,000+ users:**
- Process in chunks (1,000 at a time)
- Use async email queue (don't wait for delivery)
- Consider async database inserts
- Implement rate limiting per database
- **Total: ~1-2 hours**

---

## Monitoring & Debugging

**Check these if something goes wrong:**

1. **Validation step:** Are phone numbers being rejected? Check phone number validation logic
2. **Date parsing:** Is date format not recognized? Add to date parsing patterns
3. **Role mapping:** Is role being marked as unknown? Update role mapping dictionary
4. **Database:** Is user creation failing? Check database permissions, schema
5. **Email:** Is welcome email not sending? Check SendGrid credentials, email template
6. **Duplicates:** Are duplicates not being detected? Check duplicate query logic

**Debug mode:**
Enable logging at each step:
```javascript
console.log('Step X:', result);
```

Check n8n execution logs to see exact values at each step.

---

## Customization Points

Want to modify this workflow? Here's what to change:

**Add a new field:**
1. Add to sample data
2. Add validation rule in step 5A
3. Add transformation logic in step 6X
4. Add to database insert in step 8
5. Add to report in step 13

**Change role mapping:**
Edit `roleMapping` dictionary in step 6E

**Change email template:**
Edit email template in step 11 (or use external template service)

**Add new validation rule:**
Add condition to `validateRow()` in step 5A

**Change destination database:**
Update SQL in step 8 to target different table/database

**Add webhook notification:**
After step 15 (Slack), add new HTTP POST node

---

## Summary

The workflow is designed to be:
- **Robust** – Handles messy data and errors gracefully
- **Visible** – Detailed reporting at each stage
- **Scalable** – Works for 5 users or 5,000
- **Auditable** – Every transformation is logged
- **Maintainable** – Clear steps, easy to modify

Each step has a single responsibility, making it easy to understand and modify.
