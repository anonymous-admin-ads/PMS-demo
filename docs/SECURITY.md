# Security Architecture, Data Protection & Threat Modeling

**Document Version:** 2.0  
**Scope:** Multi-Tenant Isolation, Authentication, Insecure Direct Object References (IDOR), Document Security  

---

## 1. Threat Model & Security Posture

In a multi-owner Property Management System with a managed cloud database (Supabase PostgreSQL), the primary security threats are:
1. **Cross-Tenant Data Leakage (IDOR):** Landlord A tampering with URL parameters (e.g. `/api/v1/bills/cuid_tenant_b`) to view or manipulate Landlord B's financial records, lease contracts, or tenant contacts.
2. **Credential & Secret Leakage:** Accidental exposure of database passwords, connection strings, or privileged Supabase keys (`service_role`) in client-side code bundles or public Git repositories.
3. **In-Transit Eavesdropping:** Unencrypted database traffic between the application server and the hosted Supabase instance.
4. **Financial Manipulation:** Altering payment allocations, submitting negative payment amounts, or bypassing rent checks.
5. **Malicious File Uploads:** Uploading executables or scripts masked as tenant identity documents.
6. **Brute Force Credential Stuffing:** Automated guessing of landlord login passwords.

---

## 2. Managed Database & Secret Management (Supabase)

### 2.1 Server-Only Credential Isolation
* **Zero Client Exposure:** Database connection strings (`DATABASE_URL`, `DIRECT_URL`), PostgreSQL passwords, and any privileged Supabase API keys (`service_role`) must NEVER be exposed to the browser or prefixed with `VITE_`.
* **Git Hygiene:** All environment files (`.env`, `.env.*`) are excluded from Git via [`.gitignore`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/.gitignore). Only [`.env.example`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/.env.example) with sanitized placeholders may be committed.
* **Encrypted In-Transit Connections (TLS/SSL):** All connections to Supabase (direct and pooled) must append `?sslmode=require` to enforce TLS 1.3 encryption across the public internet.

### 2.2 Supabase Row Level Security (RLS) Review
* **Current Application Architecture:** The Express backend acts as the single authenticated intermediary, connecting to Supabase via Prisma with a dedicated database role. All authorization is enforced at the application service layer using owner-scoped SQL filters (`where: { ownerId }`).
* **Future Client Access Policy:** If direct client-side querying (e.g., via `@supabase/supabase-js` from the React SPA) is ever evaluated, **Supabase Row Level Security (RLS) MUST be explicitly enabled** on all public tables with strict `auth.uid() = owner_id` policies before enabling any client API access. Under the current Express monolith architecture, direct client access to Supabase is completely disabled.
* **Least-Privilege Principle:** Use dedicated database credentials with permissions restricted to the application schema rather than superuser accounts wherever cloud platform controls permit.

---

## 3. Authentication & Session Management

### 2.1 Password Hashing & Storage
* Passwords must NEVER be stored in plaintext.
* Hashing must use **bcrypt** with a minimum cost factor of `12` (or **Argon2id**).
* Minimum password complexity rules enforced via Zod at registration:
  * Minimum 8 characters.
  * At least one uppercase letter, one lowercase letter, one number, and one special character.

### 2.2 Token & Session Handling
* **Mechanism:** Signed JSON Web Tokens (JWT) containing `{ sub: ownerId, email }` signed with a 64+ byte secret (`HMAC-SHA256`).
* **Storage Options:**
  1. *Recommended Production:* HTTP-Only, `Secure`, `SameSite=Lax` cookies to prevent XSS exfiltration.
  2. *Standard API Bearer:* `Authorization: Bearer <token>` for programmatic API access.
* **Token Expiration:** Access tokens expire after 7 days of inactivity.

---

## 4. Strict Owner-Scoped Authorization (Anti-IDOR)

### 4.1 The Golden Rule of Multi-Tenancy
> [!IMPORTANT]
> **EVERY SQL query interacting with tenant or financial data must filter by the authenticated `ownerId`.**
> A client request must NEVER be permitted to specify `ownerId` in request bodies, query strings, or URL parameters.

### 4.2 Implementation Pattern
```typescript
// SECURE PATTERN: Tenant retrieval
export async function getTenantById(ownerId: string, tenantId: string) {
  const tenant = await prisma.tenant.findFirst({
    where: {
      id: tenantId,
      ownerId: ownerId, // Enforced constraint
      archivedAt: null
    },
    include: {
      tenancies: {
        where: { status: 'ACTIVE' },
        include: { unit: true }
      }
    }
  });

  if (!tenant) {
    // Return 404 instead of 403 to prevent record enumeration
    throw new AppError(404, 'NOT_FOUND', 'Tenant not found');
  }

  return tenant;
}
```

---

## 5. Input Validation & Injection Defenses

1. **Schema Validation:** Every API route validates incoming payloads via **Zod** before invoking service layers. Unrecognized fields are stripped.
2. **SQL Injection Defense:** All queries use Prisma ORM prepared statements and parameter binding. Raw SQL strings (`$queryRawUnsafe`) are strictly banned in business logic.
3. **Cross-Site Scripting (XSS):** React escapes HTML output by default. Any user-generated notes or strings rendered in HTML are sanitized.
4. **Negative Value & Overflow Defenses:** Monetary validation explicitly checks `amount > 0` and rejects negative values or values exceeding 12 integer digits.

---

## 6. Document & File Upload Security (Module 9)

Landlords upload sensitive personal documents (Aadhaar cards, PAN cards, lease agreements).
* **Storage Location:** Uploaded files are stored in a private directory (`uploads/`) outside the public web root (`public/`).
* **Authentication Requirement:** Files are NEVER served directly via static public URLs. Downloads must route through `/api/v1/documents/:id/download`, which verifies `Document.ownerId === req.ownerId`.
* **File Type Whitelisting:** Strictly restrict MIME types to:
  * `application/pdf`
  * `image/jpeg`
  * `image/png`
  * `image/webp`
* **File Size Cap:** Maximum 10 MB per file.
* **Filename Sanitization:** Uploaded files are renamed using cryptographically random UUIDs on disk (e.g. `doc_550e8400-e29b.pdf`), storing the original filename only as metadata.

---

## 7. Rate Limiting & Abuse Prevention

* **Authentication Routes (`/api/v1/auth/login`):** Limited to **5 requests per 15 minutes** per IP address using in-memory token bucket or `express-rate-limit`.
* **Public APIs:** Standard rate limit of 100 requests per minute per IP to protect against Denial of Service.
* **Idempotency Keys:** Payment submission endpoints accept an `Idempotency-Key` HTTP header. Duplicate requests within a 5-minute window return the existing receipt without processing duplicate charges.

---

## 8. Data Backup Protection & Disaster Recovery (Module 10)

* **Encrypted Archives:** Database backup dumps are compressed and protected with AES-256 encryption before transfer or storage.
* **Checksum Verification:** Every backup log records a `SHA-256` checksum to verify that backup files have not suffered bit-rot or tampering.
* **Access Control:** Backup creation and download endpoints are restricted exclusively to the authenticated account owner.

---

## 9. Security Testing Requirements

Before any release, the automated test suite must run explicit security checks:
1. **Cross-Owner Access Test Suite:**
   * Owner A creates Property 1 and Unit 1.
   * Owner B attempts to:
     - Read Unit 1 (`GET /api/v1/units/:id`) $\rightarrow$ Must return `404 Not Found`.
     - Update Unit 1 (`PATCH /api/v1/units/:id`) $\rightarrow$ Must return `404 Not Found`.
     - Record Payment for Unit 1 $\rightarrow$ Must return `404 Not Found`.
2. **Negative Money Attack:** Submit `{ "amount": "-500.00" }` $\rightarrow$ Must fail with `400 Bad Request`.
3. **Double Tenancy Injection:** Attempt to assign 2 active tenancies to Unit 1 simultaneously $\rightarrow$ Must fail with `409 Conflict`.

