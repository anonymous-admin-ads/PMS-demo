# Open Questions & Architecture Decision Records (ADR)

**Document Version:** 2.1  
**Status:** Living Document — Pending Client & Architecture Approvals  

This document tracks all unconfirmed business rules, design trade-offs, and client decisions required for the **Property Management System (PMS)**. Each question outlines the available options, operational implications, and recommended default decisions.

---

## 1. Summary of Decision Statuses

| # | Question Summary | Category | Impact Level | Status | Target Phase |
|---|---|---|---|---|---|
| **OQ-01** | Rent Due Date & Proration Formula | Billing / Financial | High | Pending Client Approval | Phase 4 |
| **OQ-02** | Water Bill Splitting Strategy | Utilities | High | Pending Client Approval | Phase 4 |
| **OQ-03** | Overpayment Handling Policy | Financial / Ledger | High | Pending Client Approval | Phase 4 |
| **OQ-04** | Late Fee Calculation & Application | Billing | Medium | Pending Client Approval | Phase 4 |
| **OQ-05** | Security Deposit Handling & Escrow | Accounting | Medium | Pending Client Approval | Phase 3 |
| **OQ-06** | Supabase Production Plan (Free vs Pro) | Infrastructure / Cost | High | Pending Client Approval | Phase 8 |
| **OQ-07** | Supabase Auth vs Custom Express JWT | Authentication | Medium | Decided (Express JWT for MVP) | Post-MVP |
| **OQ-08** | Supabase Storage vs S3 / Local Disk | Storage (Module 9) | Medium | Decided (Local/S3 for MVP) | Post-MVP |
| **OQ-09** | Off-Site Backup Schedule vs Supabase PITR | Operations (Module 10) | Medium | Recommended Default | Phase 6 |

---

## 2. Detailed Decision Records

### OQ-01: Rent Due Date & Proration Policy
* **Context:** When a tenant signs a lease and moves in mid-month (e.g., October 12th in a 31-day month), how should the first month's rent charge be calculated?
* **Options:**
  1. **Option A (Exact Calendar-Day Proration - RECOMMENDED):**
     $$\text{Prorated Rent} = \left(\frac{\text{Monthly Rent}}{\text{Days in Month}}\right) \times (\text{Days Active})$$
     *Implication:* Mathematically exact, fair to both parties. Requires calendar lookup in billing service.
  2. **Option B (Standardized 30-Day Commercial Proration):**
     Dividing monthly rent by 30 regardless of calendar days.
     *Implication:* Common in corporate leases; can produce minor discrepancy in 31-day or 28-day months.
  3. **Option C (Full Month Billing from Move-in Date):**
     The billing period runs from the 12th of this month to the 11th of next month.
     *Implication:* Creates staggered, non-calendar billing cycles across units, making monthly portfolio reporting confusing.
* **Architect Recommendation:** **Option A (Exact Calendar-Day Proration)**. Standard calendar-month billing (`2026-10`) is retained across all units, and mid-month move-ins receive an exact per-day charge.
* **Blocking Status:** Blocks Phase 4 charge generator implementation.

---

### OQ-02: Water Bill Splitting Rules for Shared Meters
* **Context:** Some properties have sub-meters for each unit, while other properties have a single master water meter for the entire building. How should the master bill be split among tenants?
* **Options:**
  1. **Option A (Equal Split Across Occupied Units - RECOMMENDED):**
     Master bill divided equally by the count of active tenancies during the billing period.
     *Implication:* Simple, transparent, easily understood by tenants.
  2. **Option B (Occupant-Weighted Split):**
     Bill divided based on the number of registered occupants living in each unit (e.g., family of 4 pays twice as much as a single tenant).
     *Implication:* Fairer for residential water usage, but requires landlords to strictly track tenant occupant headcounts.
  3. **Option C (Fixed Monthly Water Surcharge):**
     A fixed fee (e.g. ₹300/month) charged regardless of consumption.
     *Implication:* Does not reflect actual utility costs.
* **Architect Recommendation:** **Hybrid approach (Configurable per property)**:
  * If a unit has a `waterMeterId` $\rightarrow$ Bill by metered consumption.
  * If building has a shared meter $\rightarrow$ Default to **Option A (Equal Split per occupied unit)**, with an optional toggle for **Option B**.
* **Blocking Status:** Blocks Phase 4 Water Bill module.

---

### OQ-03: Overpayment Handling (Surplus Money)
* **Context:** Tenant owes ₹18,000 for October rent, but transfers ₹20,000 via UPI. How should the system handle the excess ₹2,000?
* **Options:**
  1. **Option A (Store as Unallocated Credit Balance - RECOMMENDED):**
     The payment of ₹20,000 is recorded. ₹18,000 is allocated to October rent (marking it `PAID`). The remaining ₹2,000 is held as an unallocated credit on the tenant's ledger, and automatically applied to November's bill when issued.
     *Implication:* Standard accounting practice. Requires the billing generator to check for existing unallocated credits when generating new bills.
  2. **Option B (Strict Validation - Reject Overpayments):**
     System rejects any payment greater than the total outstanding open charges.
     *Implication:* Frustrates landlords and tenants when money has already been transferred to the landlord's bank account.
  3. **Option C (Immediate Manual Refund Tracking):**
     System flags the surplus and prompts the landlord to issue an immediate cash/bank refund.
* **Architect Recommendation:** **Option A (Store as Unallocated Credit Balance)**.
* **Blocking Status:** Blocks Phase 4 Payment Allocation engine.

---

### OQ-04: Late Fee Policy
* **Context:** What occurs when rent is unpaid past the due date (e.g., after the 5th of the month)?
* **Options:**
  1. **Option A (Manual Landlord Application - RECOMMENDED FOR MVP):**
     Landlord can click a button to add a custom `LATE_FEE` charge to the tenant's bill if they choose to penalize them.
     *Implication:* Avoids rigid automated penalties that could strain landlord-tenant relationships.
  2. **Option B (Automated Fixed Daily Penalty):**
     System automatically adds ₹100/day starting 3 days after `dueDate`.
     *Implication:* Requires an automated recurring cron job and complex reversal workflows if the landlord chooses to waive the fee.
* **Architect Recommendation:** **Option A for MVP**, with Option B planned for post-MVP.
* **Blocking Status:** Non-blocking.

---

### OQ-05: Security Deposit Accounting
* **Context:** Should security deposits be treated as standard revenue charges or held in a separate deposit tracking ledger?
* **Options:**
  1. **Option A (Separate Deposit Field on Tenancy - RECOMMENDED):**
     Tracked via `Tenancy.securityDeposit`, `depositPaid: boolean`, and `depositReturned: Decimal`. Excluded from monthly revenue collection reports so it does not falsely inflate rental income.
  2. **Option B (Standard Charge Type `SECURITY_DEPOSIT`):**
     Treated as a regular charge on the first bill.
     *Implication:* Distorts monthly rental income reporting unless specifically filtered out.
* **Architect Recommendation:** **Option A (Dedicated Deposit Tracking)**.
* **Blocking Status:** Blocks Phase 3 Tenancy form design.

---

### OQ-06: Supabase Production Plan Selection (Free vs Pro)
* **Context:** The database is officially hosted on Supabase. Should production run on the Free Tier or Pro Tier ($25/mo)?
* **Options:**
  1. **Option A (Free Tier for Staging, Pro Tier for Production - RECOMMENDED):**
     Free tier used for development and staging. Production upgraded to Pro Tier ($25/mo) to eliminate auto-pausing after 7 days and enable daily automated backups with Point-in-Time Recovery (PITR).
  2. **Option B (Free Tier for All Environments):**
     Accepts risk of project auto-pausing if inactive for 7 days. Relies entirely on automated independent `pg_dump` cron jobs for backups.
* **Architect Recommendation:** **Option A (Pro Tier for Production)** once real landlords and tenants are onboarded.
* **Blocking Status:** Blocks Phase 8 production deployment.

---

### OQ-07: Authentication Strategy (Custom Express JWT vs Supabase Auth)
* **Context:** Supabase provides built-in Auth (GoTrue), but the application is currently architected with custom Express JWT authentication (`bcryptjs` + JWT).
* **Decision:** **Retain Custom Express JWT for MVP**.
* **Rationale:**
  * Keeps all authentication and authorization logic inside the single modular monolith.
  * Avoids vendor lock-in and avoids running client-side Supabase SDK authentication loops.
  * Supabase Auth can be re-evaluated post-MVP if OAuth/social logins or passwordless magic links are requested by the client.

---

### OQ-08: Document Storage Strategy (Local Disk / S3 vs Supabase Storage)
* **Context:** Module 9 (Documents) requires storing lease agreements, ID proofs, and receipts.
* **Decision:** **Retain Server Disk Storage with S3 Abstraction for MVP**.
* **Rationale:**
  * Uploads are secured behind the Express `/api/v1/documents/:id/download` endpoint with owner-scoped checks.
  * Zero additional cloud setup required for local development and initial testing.
  * Supabase Storage can be evaluated post-MVP if direct browser-to-bucket uploads with pre-signed URLs are required.

---

### OQ-09: Database Backup Schedule & Retention
* **Context:** Data retention rules for landlord accounting and tax audits.
* **Recommendation:**
  * Daily `pg_dump` snapshots retained for **30 days**.
  * Weekly snapshots retained for **12 weeks**.
  * Year-end snapshots retained for **7 years** (Indian Income Tax Act audit requirement).
* **Blocking Status:** Non-blocking for early phases.
