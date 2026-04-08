# LoanAssist Lite - Security Implementation Guide

## Overview
This document explains the security architecture of LoanAssist Lite, including authentication, data protection, role-based access control (RBAC), and audit logging.

---

## 1. Authentication & Session Management

### Current Implementation (Frontend Demo)
- **Session Storage**: Sessions stored in `localStorage` with expiration timestamps
- **Session Format**:
```typescript
{
  id: string;           // Unique user ID
  name: string;         // Username
  role: UserRole;       // Role assignment
  expiresAt: number;    // Expiration timestamp (ms)
}
```

- **Session Duration**: Default 30 minutes, configurable in Settings
- **Validation**: `getSession()` validates expiry on every access

### Production Requirements
- ✅ **TLS/HTTPS**: All traffic must be encrypted
- ✅ **Backend Authentication**: 
  - POST `/api/auth/login` with username + password
  - Server validates credentials against salted, hashed passwords
  - Returns secure session token (JWT or opaque token)
- ✅ **Secure Cookies**: 
  - `HttpOnly` flag (JavaScript cannot read)
  - `Secure` flag (HTTPS only)
  - `SameSite=Strict` (CSRF protection)
- ✅ **Password Requirements**:
  - Minimum 12 characters
  - Must include uppercase, lowercase, numbers, special chars
  - Hashed with bcrypt or Argon2 (minimum 10 rounds)
  - Never logged or displayed
- ✅ **Multi-Factor Authentication (MFA)**:
  - Time-based One-Time Password (TOTP)
  - SMS or email verification codes
  - Required for sensitive roles (admin, underwriter, collections)

---

## 2. Role-Based Access Control (RBAC)

### Roles & Permissions
```typescript
'applicant'         → Can only apply for loans, view own applications
'loan_officer'      → Review applications, recommend decisions
'underwriter'       → Approve/reject applications, manage exceptions
'collections'       → Record payments, manage delinquencies
'branch_manager'    → Oversee portfolio, approve exceptions
'admin'             → Configure system, manage users, products
'auditor'           → View-only access to audit logs and transactions
```

### Access Control Implementation

**Frontend Route Protection** (Current):
```typescript
createAuthLoader(['applicant']) // Only applicant role can access
createAuthLoader(['admin', 'branch_manager']) // Multiple roles
```

**Navigation Filtering** (Current):
```typescript
navItems.filter((item) => user && item.roles.includes(user.role))
```

**Backend API Protection** (Required):
```typescript
// Every API endpoint must verify:
1. User is authenticated (valid session token)
2. User's role has permission for this endpoint
3. User can only access their own data or assigned records
```

### Example: Loan Application Endpoint
```typescript
GET /api/applications/:id
- Verify: user is authenticated
- Verify: user is applicant OR (user is staff AND has permission)
- Verify: if applicant, can only see own applications
- Return: 403 Forbidden if unauthorized
```

---

## 3. Data Protection

### Encryption at Rest
**Sensitive Fields to Encrypt**:
- National ID / SSN
- Banking account numbers
- Salary/income information
- Personal contact details
- Document content

**Implementation**:
```sql
-- Database schema
CREATE TABLE users (
  id UUID PRIMARY KEY,
  username VARCHAR(255) NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  email VARCHAR(255) ENCRYPTED,      -- AES-256
  phone VARCHAR(255) ENCRYPTED,      -- AES-256
  created_at TIMESTAMP
);

CREATE TABLE applicants (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  national_id VARCHAR(255) ENCRYPTED,  -- AES-256
  ssn VARCHAR(255) ENCRYPTED,          -- AES-256
  dob DATE ENCRYPTED,                  -- AES-256
  address TEXT ENCRYPTED,              -- AES-256
  contact_info JSONB ENCRYPTED         -- AES-256
);
```

### Encryption in Transit
- **TLS 1.3+**: All HTTP traffic encrypted
- **Certificate**: Valid, non-expired SSL/TLS certificate from trusted CA
- **HSTS**: Enforce HTTPS for at least 1 year

### Data Masking in Logs
```typescript
// ❌ NEVER log passwords, tokens, or full PII
console.log(password);           // WRONG
console.log(sessionToken);       // WRONG
console.log(fullNationalId);    // WRONG

// ✅ Log safely with masking
console.log(`User ${userId} authenticated`);
console.log(`National ID: ${nationalId.slice(-4)}`);
console.log(`Payment of $${amount} processed`);
```

---

## 4. Application Data Flow

### Data Storage (Current - Demo)
```
┌─────────────────┐
│  User Input     │
│  (Form Data)    │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────┐
│  applicationData.ts             │
│  - saveApplication()            │
│  - getApplications()            │
│  - updateApplicationStatus()    │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────┐
│  localStorage   │
│  (Browser)      │
└─────────────────┘
```

### Data Storage (Production Required)
```
┌─────────────────────────────────┐
│  Frontend (React)                │
│  - Validates input              │
│  - Encrypts PII before send      │
└────────┬────────────────────────┘
         │ TLS/HTTPS
         ▼
┌──────────────────────────────────┐
│  Backend API (Node.js/Python)     │
│  - Authenticates request         │
│  - Verifies role/permissions     │
│  - Validates & sanitizes input   │
│  - Logs action to audit log      │
└────────┬─────────────────────────┘
         │
         ▼
┌──────────────────────────────────┐
│  Database (PostgreSQL/MySQL)      │
│  - Encrypted sensitive fields    │
│  - Transactional integrity       │
│  - Referential integrity         │
│  - Backups encrypted & secured   │
└──────────────────────────────────┘
```

---

## 5. Audit Logging

### What to Log
```typescript
interface AuditLog {
  id: string;
  userId: string;
  action: string;              // 'create', 'update', 'delete', 'approve', 'reject'
  resource: string;            // 'application', 'payment', 'user'
  resourceId: string;
  timestamp: Date;
  ipAddress: string;           // User's IP (for security monitoring)
  changes: {
    before: any;               // Previous values
    after: any;                // New values
  };
  status: 'success' | 'failure';
  details?: string;            // Additional context
}
```

### Audit Log Examples
```typescript
// ✅ Application submitted
{
  userId: 'user_123',
  action: 'create',
  resource: 'application',
  resourceId: 'app_001',
  timestamp: '2026-04-08T10:30:00Z',
  changes: {
    after: {
      applicantName: 'John Doe',
      amount: 5000,
      status: 'pending'
    }
  }
}

// ✅ Application approved
{
  userId: 'underwriter_456',
  action: 'update',
  resource: 'application',
  resourceId: 'app_001',
  timestamp: '2026-04-09T14:15:00Z',
  changes: {
    before: { status: 'pending' },
    after: { status: 'approved' }
  }
}
```

### Immutability
- Audit logs must be **append-only**
- Cannot be modified or deleted (database constraints)
- Regular exports for external archival

---

## 6. Document & Evidence Storage

### File Handling
```typescript
interface StoredDocument {
  id: string;
  ownerType: 'applicant' | 'loan';
  ownerId: string;
  docType: 'id' | 'address_proof' | 'income_proof';
  fileName: string;
  fileHash: string;           // SHA-256
  fileURI: string;            // Local path or S3 URI
  uploadedBy: string;         // User ID
  uploadedAt: Date;
  verifiedBy?: string;        // Auditor user ID
  verifiedAt?: Date;
  status: 'pending' | 'verified' | 'rejected';
}
```

### Security Measures
- **File Type Validation**: Accept only PDF, PNG, JPG
- **File Size Limit**: 10MB per document
- **Virus Scanning**: Scan uploaded files with ClamAV
- **Hash Verification**: SHA-256 hash stored for tamper detection
- **Access Control**: Only authorized staff can view documents
- **Retention**: Delete after statute of limitations (typically 7-10 years)

---

## 7. API Security

### Request Validation
```typescript
// Sanitize input to prevent SQL injection and XSS
const sanitize = (input: string) => {
  return input
    .trim()
    .replace(/[^\w\s-]/g, '') // Remove special chars
    .substring(0, 255);        // Limit length
};

// Validate email format
const isValidEmail = (email: string) => {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
};

// Validate phone format
const isValidPhone = (phone: string) => {
  return /^\d{10}$/.test(phone.replace(/\D/g, ''));
};
```

### Rate Limiting
```typescript
// Prevent brute-force attacks
// Example: Max 5 login attempts per 15 minutes per IP

// Prevent DoS attacks
// Example: Max 100 requests per minute per user
```

### CORS (Cross-Origin Resource Sharing)
```typescript
// Only allow requests from trusted domains
const corsOptions = {
  origin: ['https://app.loanassist.local'],
  credentials: true,
  optionsSuccessStatus: 200
};
```

---

## 8. Password & Credential Management

### Password Policy
```
✅ Minimum 12 characters
✅ Must include: uppercase, lowercase, numbers, special chars
✅ Cannot contain username or common patterns
✅ Expires every 90 days
✅ Cannot reuse last 5 passwords
✅ Account locked after 5 failed attempts (15 min lockout)
```

### Secure Password Storage
```python
# Python example with bcrypt
import bcrypt

password = "user_input_password"
salt = bcrypt.gensalt(rounds=12)
password_hash = bcrypt.hashpw(password.encode(), salt)

# Verify during login
if bcrypt.checkpw(input_password.encode(), stored_hash):
    # Password is correct
    pass
```

### Secret Management
```typescript
// Store API keys, database credentials in environment variables
// Never commit secrets to version control

// .env (NEVER commit this)
DATABASE_PASSWORD=super_secret_password_123
JWT_SECRET=another_secret_key_456
ENCRYPTION_KEY=encryption_key_789

// Access in code
const dbPassword = process.env.DATABASE_PASSWORD;
```

---

## 9. Incident Response

### Security Incident Process
1. **Detect**: Monitor logs for suspicious activity
2. **Assess**: Determine scope and impact
3. **Contain**: Isolate affected systems
4. **Eradicate**: Remove threat/vulnerability
5. **Recover**: Restore systems to normal state
6. **Review**: Post-incident analysis

### Examples
- **Account Compromise**: Reset password, invalidate sessions, audit activity
- **Data Breach**: Notify affected users, review logs, patch vulnerabilities
- **Failed Authentication Attempts**: Lock account, notify user, alert admin

---

## 10. Compliance & Regulations

### Standards
- **ISO 27001**: Information Security Management
- **PCI DSS**: Payment Card Industry (if accepting payments)
- **GDPR**: Data protection (if EU users)
- **CCPA**: Consumer privacy (if California users)
- **Local Financial Regulations**: Microfinance licensing

### Data Retention
```
Users:            Keep indefinitely (unless GDPR delete request)
Applications:     Keep 7 years minimum (financial record)
Payments:         Keep 7 years minimum
Documents:        Keep 10 years minimum
Audit Logs:       Keep 10 years minimum
```

---

## 11. Deployment Security Checklist

### Before Going Live
- [ ] Enable HTTPS/TLS 1.3+
- [ ] Configure firewall rules
- [ ] Set up WAF (Web Application Firewall)
- [ ] Enable database encryption
- [ ] Configure backups (encrypted, off-site)
- [ ] Set up monitoring & alerting
- [ ] Enable audit logging
- [ ] Implement rate limiting
- [ ] Configure CORS properly
- [ ] Set up environment variables
- [ ] Disable debug mode in production
- [ ] Enable security headers (CSP, X-Frame-Options, etc.)
- [ ] Set up DDoS protection
- [ ] Enable intrusion detection (IDS)
- [ ] Configure VPN for staff access

### Ongoing
- [ ] Weekly security updates
- [ ] Monthly vulnerability scans
- [ ] Quarterly penetration testing
- [ ] Annual security audit
- [ ] Regular backups & restore tests
- [ ] Staff security training

---

## 12. Current Implementation Status

### ✅ Implemented (Frontend)
- Session-based authentication with expiration
- Role-based route protection
- Session validation on each access
- Navigation filtering by role
- Basic form validation
- Application data storage

### ⚠️ Missing (Must Add for Production)
- Backend API authentication
- Password hashing & verification
- HTTPS/TLS enforcement
- Database encryption
- Audit logging
- Input sanitization
- Rate limiting
- Multi-factor authentication
- Secure document storage
- API key management

---

## Quick Start: Adding Security to Your Backend

### 1. Setup Database with Encryption
```sql
-- Create users table with hashed passwords
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  username VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  email VARCHAR(255) ENCRYPTED NOT NULL,
  role VARCHAR(50) NOT NULL,
  status VARCHAR(50) DEFAULT 'active',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_login TIMESTAMP
);

-- Create audit log table
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES users(id),
  action VARCHAR(100) NOT NULL,
  resource VARCHAR(100) NOT NULL,
  resource_id UUID,
  timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  ip_address INET,
  changes JSONB,
  status VARCHAR(50)
);
```

### 2. Setup Authentication API
```typescript
// POST /api/auth/login
app.post('/api/auth/login', async (req, res) => {
  const { username, password } = req.body;

  // 1. Validate input
  if (!username || !password) {
    return res.status(400).json({ error: 'Missing credentials' });
  }

  // 2. Find user
  const user = await db.query(
    'SELECT id, password_hash, role FROM users WHERE username = $1',
    [username]
  );

  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // 3. Verify password
  const isValid = await bcrypt.compare(password, user.password_hash);
  if (!isValid) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // 4. Create session token (JWT)
  const token = jwt.sign(
    { userId: user.id, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: '30m' }
  );

  // 5. Log authentication
  await auditLog(user.id, 'login', 'user', user.id, 'success');

  // 6. Return token in secure cookie
  res.cookie('sessionToken', token, {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: 30 * 60 * 1000
  });

  res.json({ message: 'Logged in successfully' });
});
```

---

## Support & Questions
For security-related questions, consult:
- OWASP Top 10: https://owasp.org/www-project-top-ten/
- CWE/SANS Top 25: https://cwe.mitre.org/top25/
- Your local financial regulator
- Security consultant/auditor
