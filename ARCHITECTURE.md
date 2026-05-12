# Bulk Email Management Portal - Technical Architecture Document

## 1. System Overview

The Bulk Email Management Portal is a PHP-based web application designed to manage bulk email campaigns. It provides functionality for contact management, email template creation, campaign scheduling, and real-time delivery tracking.

### 1.1 Core Capabilities
- Contact import via CSV
- Dynamic email template management with variable substitution
- Campaign creation with immediate or scheduled sending
- Background email processing with retry logic
- Real-time campaign statistics via AJAX polling
- Email delivery logging and tracking

---

## 2. Architecture Pattern

**Pattern**: MVC-inspired procedural architecture with separation of concerns

### 2.1 Layers
- **Presentation Layer**: PHP pages with embedded HTML/Bootstrap UI
- **Business Logic Layer**: `functions.php` utility functions
- **Data Access Layer**: PDO-based database interactions
- **Background Processing Layer**: Cron-based email worker

---

## 3. Technology Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Backend Language | PHP | 7.x+ |
| Database | MySQL | 5.7+ |
| Database Driver | PDO | Native |
| Email Library | PHPMailer | ^7.0 |
| Frontend Framework | Bootstrap | 5.3.3 |
| JavaScript Library | jQuery | 3.7.1 |
| Dependency Manager | Composer | Latest |
| Web Server | MAMP (Apache) | Latest |

---

## 4. Database Schema

### 4.1 Entity Relationship Diagram (Conceptual)

```
contacts (1) ----< (M) campaign_recipients (M) >---- (1) campaigns
                                                            |
                                                            | (1)
                                                            |
                                                            v
                                                      email_templates (1)
                                                            
email_logs (M) >---- (1) campaigns
```

### 4.2 Table Definitions

#### **contacts**
Stores all imported contact records.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | INT | PRIMARY KEY, AUTO_INCREMENT | Unique contact identifier |
| name | VARCHAR(255) | | Contact full name |
| email | VARCHAR(255) | UNIQUE | Contact email address |
| company | VARCHAR(255) | | Company name |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | Record creation time |

**Indexes**: UNIQUE on `email`

---

#### **email_templates**
Stores reusable email templates with variable placeholders.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | INT | PRIMARY KEY, AUTO_INCREMENT | Template identifier |
| title | VARCHAR(255) | | Template name |
| subject | VARCHAR(255) | | Email subject line |
| body | LONGTEXT | | HTML email body with {{variables}} |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | Template creation time |

**Supported Variables**: `{{name}}`, `{{company}}`, `{{email}}`

---

#### **campaigns**
Stores campaign metadata and status.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | INT | PRIMARY KEY, AUTO_INCREMENT | Campaign identifier |
| name | VARCHAR(255) | | Campaign name |
| template_id | INT | FOREIGN KEY → email_templates.id | Associated template |
| status | ENUM | 'pending', 'processing', 'completed' | Campaign state |
| send_type | VARCHAR(50) | 'now', 'scheduled' | Send mode |
| scheduled_at | DATETIME | NULL | Scheduled send time |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | Campaign creation time |

**Status Flow**: `pending` → `processing` → `completed`

---

#### **campaign_recipients**
Tracks per-recipient email delivery status.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | INT | PRIMARY KEY, AUTO_INCREMENT | Recipient record ID |
| campaign_id | INT | FOREIGN KEY → campaigns.id | Parent campaign |
| contact_id | INT | FOREIGN KEY → contacts.id | Target contact |
| email | VARCHAR(255) | | Recipient email (denormalized) |
| status | ENUM | 'pending', 'sent', 'failed' | Delivery status |
| retry_count | INT | DEFAULT 0 | Failed attempt counter |
| sent_at | DATETIME | NULL | Successful send timestamp |

**Indexes**: Composite on `(campaign_id, status)` for efficient queue queries

---

#### **email_logs**
Audit trail for all email operations.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | INT | PRIMARY KEY, AUTO_INCREMENT | Log entry ID |
| campaign_id | INT | FOREIGN KEY → campaigns.id | Associated campaign |
| email | VARCHAR(255) | | Target email address |
| status | VARCHAR(50) | | Operation status |
| message | TEXT | | Status message or error |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | Log timestamp |

---

## 5. Application Components

### 5.1 Frontend Pages

#### **index.php** (Dashboard)
- **Purpose**: Main landing page with system overview
- **Features**:
  - Dashboard cards (total contacts, templates, campaigns)
  - Paginated contacts table
  - Real-time campaign stats via AJAX
- **Key Logic**:
  - Aggregation queries for dashboard metrics
  - Pagination with LIMIT/OFFSET
  - jQuery polling every 3 seconds to `api/campaign_stats.php`

---

#### **upload_contacts.php** (Contact Import)
- **Purpose**: CSV file upload and contact import
- **Features**:
  - CSV parsing with header skip
  - Email validation via `FILTER_VALIDATE_EMAIL`
  - Duplicate detection by email
- **CSV Format**:
  ```
  name,email,company
  John Doe,john@example.com,ABC Corp
  ```
- **Error Handling**: Skips invalid emails, prevents duplicates

---

#### **templates.php** (Template Management)
- **Purpose**: Create and list email templates
- **Features**:
  - WYSIWYG-ready textarea for HTML body
  - Variable placeholder support (`{{name}}`, `{{company}}`, `{{email}}`)
  - Paginated template list
  - Edit/Delete actions
- **Validation**: All fields required

---

#### **campaigns.php** (Campaign Management)
- **Purpose**: Create and monitor campaigns
- **Features**:
  - Campaign creation form with template selection
  - Send type: Immediate (`now`) or Scheduled (`scheduled`)
  - Dynamic schedule input (datetime-local)
  - Campaign list with status badges
  - "Queue Emails" action for pending campaigns
  - Auto-refresh worker via AJAX (every 10 seconds)
- **Status Indicators**:
  - Pending (yellow), Processing (blue), Completed (green)

---

#### **send_campaign.php** (Campaign Queue Builder)
- **Purpose**: Populate `campaign_recipients` table
- **Workflow**:
  1. Validate campaign exists and is not already processed
  2. Check scheduled time (if applicable)
  3. Fetch all contacts
  4. Prevent duplicate recipient records
  5. Insert pending recipients
  6. Log queue operations
  7. Update campaign status to `processing`
- **Safeguards**: Duplicate prevention, status validation

---

### 5.2 Backend Components

#### **config/database.php** (Database Connection)
- **Purpose**: PDO connection singleton
- **Configuration**:
  - Windows: `localhost:3306`, user: `root`, password: ``
  - macOS: `localhost:8889`, user: `root`, password: `root`
- **Settings**: UTF-8 charset, exception error mode

---

#### **functions.php** (Utility Library)
Core helper functions:

| Function | Purpose |
|----------|---------|
| `clean($data)` | Sanitize input with htmlspecialchars |
| `isValidEmail($email)` | Email validation |
| `replaceTemplateVariables($body, $contact)` | Template variable substitution |
| `redirect($url)` | HTTP redirect helper |
| `setFlashMessage($type, $message)` | Session-based flash messages |
| `displayFlashMessage()` | Render flash messages |
| `randomString($length)` | Generate random strings |
| `formatDate($date)` | Date formatting helper |

---

#### **cron/send_emails.php** (Email Worker)
- **Purpose**: Background email processing daemon
- **Execution**: Cron job or AJAX polling (every 10 seconds)
- **Processing Logic**:
  1. **Query**: Fetch up to 3 recipients with:
     - Status: `pending` or `failed`
     - Retry count < 3
     - Campaign is due (for scheduled campaigns)
  2. **Lock**: Update status to `processing` to prevent race conditions
  3. **Send**: Use PHPMailer with Gmail SMTP
  4. **Success**: Mark as `sent`, log success
  5. **Failure**: Increment retry count, mark as `failed` after 3 attempts
  6. **Completion Check**: Mark campaign as `completed` when all emails sent

**Concurrency Control**: Row-level locking via status update

**PHPMailer Configuration**:
```php
Host: smtp.gmail.com
Port: 587
Encryption: STARTTLS
Auth: Gmail credentials (user-provided)
```

---

#### **api/campaign_stats.php** (Stats API)
- **Purpose**: Real-time campaign statistics
- **Response Format** (JSON):
  ```json
  {
    "sent": 150,
    "pending": 25,
    "failed": 5
  }
  ```
- **Query**: Aggregation on `campaign_recipients` by status

---

## 6. Data Flow Diagrams

### 6.1 Campaign Creation Flow

```
User → campaigns.php (Create Form)
  ↓
POST /campaigns.php
  ↓
INSERT INTO campaigns (status='processing' if now, 'pending' if scheduled)
  ↓
Redirect to campaigns.php
  ↓
User clicks "Queue Emails"
  ↓
send_campaign.php
  ↓
INSERT INTO campaign_recipients (all contacts)
  ↓
UPDATE campaigns SET status='processing'
  ↓
Redirect to campaigns.php
```

---

### 6.2 Email Sending Flow

```
Cron/AJAX Trigger → cron/send_emails.php
  ↓
SELECT pending/failed recipients (LIMIT 3)
  ↓
FOR EACH recipient:
  ↓
  UPDATE status='processing' (lock)
  ↓
  PHPMailer → Gmail SMTP
  ↓
  Success?
    ├─ YES → UPDATE status='sent', sent_at=NOW()
    │         INSERT INTO email_logs (status='success')
    │
    └─ NO  → UPDATE retry_count++
              IF retry_count >= 3:
                UPDATE status='failed'
              INSERT INTO email_logs (status='failed')
  ↓
Check if all recipients sent
  ↓
  YES → UPDATE campaigns SET status='completed'
```

---

### 6.3 Real-Time Stats Flow

```
index.php (Dashboard)
  ↓
jQuery setInterval (3 seconds)
  ↓
AJAX GET /api/campaign_stats.php?campaign_id=1
  ↓
SELECT SUM(status='sent'), SUM(status='pending'), SUM(status='failed')
  ↓
JSON Response
  ↓
Update DOM (#sent, #pending, #failed)
```

---

## 7. Security Considerations

### 7.1 Implemented Measures
- **SQL Injection Prevention**: Prepared statements with PDO
- **XSS Prevention**: `htmlspecialchars()` on all output
- **Email Validation**: `FILTER_VALIDATE_EMAIL` filter
- **Session Management**: Flash messages via PHP sessions
- **CSRF Protection**: ❌ **NOT IMPLEMENTED** (Recommendation: Add CSRF tokens)

### 7.2 Recommendations
1. **Add CSRF tokens** to all forms
2. **Implement authentication** (currently no login system)
3. **Use environment variables** for database credentials
4. **Enable HTTPS** for production
5. **Rate limiting** on email sending
6. **Input sanitization** on CSV uploads (prevent CSV injection)
7. **Secure Gmail credentials** (use App Passwords, not plain passwords)

---

## 8. Performance Optimization

### 8.1 Current Optimizations
- **Pagination**: Limits query results (5 per page)
- **Batch Processing**: Processes 3 emails per cron run
- **Indexed Queries**: UNIQUE index on `contacts.email`

### 8.2 Scalability Recommendations
1. **Queue System**: Replace cron with Redis/RabbitMQ
2. **Database Indexing**: Add composite indexes on `(campaign_id, status)`
3. **Connection Pooling**: Use persistent PDO connections
4. **Caching**: Cache dashboard metrics with Redis
5. **Async Processing**: Use PHP-FPM with background workers
6. **CDN**: Serve Bootstrap/jQuery from CDN (already implemented)

---

## 9. Error Handling & Logging

### 9.1 Error Handling Strategy
- **Database Errors**: Try-catch blocks with PDO exceptions
- **Email Failures**: Logged to `email_logs` table
- **Retry Logic**: Up to 3 attempts per recipient
- **User Feedback**: Session-based flash messages

### 9.2 Logging Mechanisms
- **email_logs Table**: Audit trail for all email operations
- **Console Output**: `echo` statements in cron worker
- **PHP Error Logs**: Disabled in production (commented out)

---

## 10. Deployment Guide

### 10.1 Prerequisites
- PHP 7.4+
- MySQL 5.7+
- Composer
- MAMP/XAMPP/LAMP stack

### 10.2 Installation Steps

1. **Clone/Extract Project**
   ```bash
   cd /Applications/MAMP/htdocs/
   # Extract project files
   ```

2. **Install Dependencies**
   ```bash
   cd my_php_project
   composer install
   ```

3. **Create Database**
   ```bash
   mysql -u root -p < schema.sql
   ```

4. **Configure Database**
   Edit `config/database.php`:
   ```php
   $host = "localhost";
   $port = "8889"; // 3306 for Windows
   $dbname = "bulk_email_portal";
   $username = "root";
   $password = "root"; // Empty for Windows
   ```

5. **Configure Email**
   Edit `cron/send_emails.php`:
   ```php
   $mail->Username = 'your_email@gmail.com';
   $mail->Password = 'your_app_password'; // Use Gmail App Password
   ```

6. **Set Up Cron Job** (Production)
   ```bash
   */1 * * * * php /path/to/cron/send_emails.php >> /var/log/email_worker.log 2>&1
   ```

7. **Start Server**
   ```bash
   # MAMP: Start Apache and MySQL
   # Access: http://localhost:8888/my_php_project/
   ```

---

## 11. Testing Strategy

### 11.1 Manual Testing Checklist
- [ ] Upload CSV with valid/invalid emails
- [ ] Create template with variables
- [ ] Create immediate campaign
- [ ] Create scheduled campaign
- [ ] Verify email delivery
- [ ] Check retry logic (force failure)
- [ ] Monitor real-time stats
- [ ] Test pagination on all pages

### 11.2 Test Data
Sample CSV:
```csv
name,email,company
Test User,test@example.com,Test Corp
Invalid User,invalid-email,Bad Corp
```

---

## 12. Known Limitations

1. **No Authentication**: Anyone can access the system
2. **No CSRF Protection**: Forms vulnerable to CSRF attacks
3. **Hardcoded Credentials**: Database/email credentials in code
4. **Single Campaign Stats**: Dashboard only shows campaign ID 1
5. **No Email Throttling**: Could trigger Gmail rate limits
6. **No File Upload Validation**: CSV uploads not sanitized
7. **No Template Preview**: Cannot preview emails before sending
8. **No Unsubscribe Mechanism**: No opt-out functionality

---

## 13. Future Enhancements

### 13.1 Short-Term
- [ ] Add user authentication (login/logout)
- [ ] Implement CSRF protection
- [ ] Add email preview functionality
- [ ] Support multiple SMTP providers (SendGrid, Mailgun)
- [ ] Add contact segmentation/tagging

### 13.2 Long-Term
- [ ] REST API for external integrations
- [ ] Webhook support for delivery notifications
- [ ] A/B testing for email templates
- [ ] Advanced analytics dashboard
- [ ] Multi-tenant support
- [ ] Email bounce handling
- [ ] Unsubscribe link management

---

## 14. Maintenance & Monitoring

### 14.1 Regular Maintenance Tasks
- Monitor `email_logs` for failures
- Clean up old campaign data
- Backup database weekly
- Update PHPMailer library
- Review Gmail sending limits

### 14.2 Monitoring Metrics
- Email delivery rate (sent/total)
- Average send time per email
- Retry rate (failed/total)
- Campaign completion time
- Database table sizes

---

## 15. Support & Documentation

### 15.1 Key Files
- `README.md`: Project overview and setup
- `ARCHITECTURE.md`: This document
- `schema.sql`: Database schema
- `composer.json`: PHP dependencies

### 15.2 External Resources
- [PHPMailer Documentation](https://github.com/PHPMailer/PHPMailer)
- [Bootstrap 5 Docs](https://getbootstrap.com/docs/5.3/)
- [PDO Documentation](https://www.php.net/manual/en/book.pdo.php)

---

## 16. Glossary

| Term | Definition |
|------|------------|
| Campaign | A bulk email sending operation targeting multiple contacts |
| Template | Reusable email content with variable placeholders |
| Recipient | Individual contact in a campaign queue |
| Queue | List of pending emails to be sent |
| Retry Logic | Automatic re-attempt mechanism for failed emails |
| Flash Message | Temporary session-based user notification |
| Cron Worker | Background script for email processing |

---

**Document Version**: 1.0  
**Last Updated**: 2024  
**Author**: System Architect  
**Status**: Production Ready
