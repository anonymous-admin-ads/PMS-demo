# Milestones, Quality Criteria & Acceptance Checklist

**Document Version:** 2.0  
**Status:** Approved Acceptance Framework  

---

## 1. Quality Gates & Measurable Criteria by Phase

| Phase | Milestone Name | Measurable Acceptance Criteria |
|---|---|---|
| **Phase 0** | Baseline & Architecture | All distributed documentation and Prisma schema created, reviewed, and configured for Supabase managed PostgreSQL (`DATABASE_URL` pooler and `DIRECT_URL` direct). |
| **Phase 1** | Scaffolding & CI | `apps/web` and `apps/api` scaffolded; `pnpm typecheck`, `pnpm lint`, and CI pass without warnings. |
| **Phase 2** | Database & Auth | Supabase project connected via SSL; Prisma migrations applied successfully via `DIRECT_URL`; bcrypt/JWT auth working; cross-owner test denies access 100% of the time. |
| **Phase 3** | Properties & Tenancies | Properties, units, and tenants created via UI; active occupancy concurrency guard rejects overlapping tenancies. |
| **Phase 4** | Billing & Payments Engine | Rent/water charges generated; duplicate `periodKey` charges rejected; partial payments allocated via FIFO in atomic transaction. |
| **Phase 5** | Ledger & Dashboard | Running ledger reconciles debits/credits; executive dashboard calculates occupancy and arrears from real DB queries. |
| **Phase 6** | Reports & Operations | CSV report downloads RFC 4180 compliant; documents upload and stream securely; DB backup dumps verified. |
| **Phase 7** | Mobile UI & Demo | All screens verified on 375px mobile viewport; demo dataset seeded with 3 properties, 15 units, 10 tenants. |
| **Phase 8** | Final Sign-Off | All 13 master acceptance workflows pass without human workarounds. |

---

## 2. Core Domain Acceptance Criteria

### 2.1 Financial Correctness Criteria
1. **Decimal Precision:** All monetary amounts are stored as `Decimal(12, 2)` in PostgreSQL and handled with decimal arithmetic. Binary floating-point math (`0.1 + 0.2 === 0.30000000000000004`) is strictly forbidden.
2. **FIFO Allocation:** Payments must settle oldest unpaid charges first by `dueDate ASC`.
3. **Partial Payments:** If a tenant owes ₹18,500 and pays ₹10,000:
   * Payment allocation records ₹10,000 to the charge.
   * Charge status updates to `PARTIALLY_PAID`.
   * Remaining charge balance reflects exactly ₹8,500.00.
4. **Ledger Reconciliation:** At all times, the tenant's ledger balance must equal:
   $$\sum \text{Charges} - \sum \text{Allocations}$$

### 2.2 Security & Multi-Tenancy Criteria
1. **Zero Client Trust:** The server derives `ownerId` exclusively from the verified session token. Any `ownerId` sent in request bodies or query parameters is discarded.
2. **Cross-Owner Isolation:** When Owner B attempts to access Owner A's property, unit, tenant, bill, or document by ID:
   * API returns `404 Not Found` (preferred to avoid ID enumeration) or `403 Forbidden`.
   * No data belonging to Owner A is ever returned to Owner B.
3. **Password Security:** Passwords hashed with bcrypt (cost $\ge 12$). Plaintext passwords or hashes are never returned in API responses.

### 2.3 Responsive UI Criteria
1. **Mobile Ergonomics:** On 375px–640px screens, all action buttons meet the 44px touch target standard.
2. **Table Responsiveness:** Data tables transform into stacked, readable cards on mobile viewports.
3. **Zero Horizontal Layout Shift:** No unstyled text, horizontal scrollbars on page roots, or broken cards.

---

## 3. Master Final Acceptance Checklist (13 Core Workflows)

Before the PMS is deemed production-ready, the following 13 verification workflows must be executed and signed off:

- [ ] **Workflow 1: Property and Unit Setup**
  * Owner creates a residential property ("Greenwood Residency") and an independent villa.
  * Owner adds 5 units with monthly rent (e.g. ₹15,000/mo) and security deposit specs.
  * Duplicate unit numbers within the same property are rejected with a clear validation error.

- [ ] **Workflow 2: Tenant Intake and Tenancy Assignment**
  * Owner creates a tenant profile (Name, Phone, Aadhaar info).
  * Owner assigns the tenant to Unit 101 with move-in date and agreed rent.
  * System marks Unit 101 as `OCCUPIED`.
  * Attempting to assign a second active tenant to Unit 101 is rejected with a concurrency conflict error.

- [ ] **Workflow 3: Monthly Rent Charge Generation**
  * Owner generates monthly bills for the current period (`2026-11`).
  * System creates an itemized `Bill` and `Charge` with `periodKey = "2026-11-RENT"`.
  * Running the generator a second time is idempotent and rejects duplicate charge creation.

- [ ] **Workflow 4: Partial Payment Recording**
  * Tenant owes ₹18,500. Owner records a partial UPI payment of ₹10,000 with a transaction reference.
  * System assigns a sequential receipt number (e.g. `REC-2026-0001`).
  * The payment is allocated to the charge; charge status updates to `PARTIALLY_PAID`.

- [ ] **Workflow 5: Balance and Ledger Reconciliation**
  * Owner opens the tenant's Ledger view.
  * Ledger displays: Charge (+₹18,500.00), Payment (-₹10,000.00), Outstanding Balance: ₹8,500.00.
  * All totals match the bill summary exactly.

- [ ] **Workflow 6: Payment Completion and Bill Settlement**
  * Owner records a second payment of ₹8,500.00 (Cash).
  * System generates receipt `REC-2026-0002`.
  * Charge status updates to `PAID`. Bill status updates to `PAID`. Tenant ledger balance becomes ₹0.00.

- [ ] **Workflow 7: Move-Out and Unit Reversion**
  * Owner records a move-out for the tenant with a departure date.
  * Tenancy status transitions to `ENDED`.
  * Unit 101 status automatically reverts to `AVAILABLE` for future assignments.

- [ ] **Workflow 8: Historical Audit Trail Preservation**
  * After move-out, owner navigates to historical reports and previous bills.
  * All past invoices, receipts, payment allocations, and ledger entries for the former tenant remain fully accessible and unchanged.

- [ ] **Workflow 9: Cross-Owner Authorization Attack Defense**
  * An automated test suite simulates Owner B attempting to read, update, or delete Owner A's units, tenants, and bills.
  * All requests return `404 Not Found` or `403 Forbidden`. Zero data leakage occurs.

- [ ] **Workflow 10: Report Generation and CSV Export**
  * Owner navigates to Reports, viewing Monthly Collections, Overdue Arrears, and Occupancy.
  * All aggregated numbers match underlying ledger rows.
  * Owner clicks *Export to CSV*, downloading an RFC 4180-compliant file that opens cleanly in spreadsheet tools.

- [ ] **Workflow 11: Secure Document Storage and Retrieval**
  * Owner uploads a PDF lease agreement and tenant ID proof.
  * File is stored outside public web roots with a random hash filename.
  * File download is verified to require an authenticated session matching the owning landlord.

- [ ] **Workflow 12: Backup Restoration in an Isolated Test Environment**
  * Owner triggers an on-demand database backup.
  * System generates a compressed dump and records a SHA-256 checksum in `BackupLog`.
  * The dump file is restored into a clean test database (`pms_restore_test`).
  * Row counts and financial totals match 100%.

- [ ] **Workflow 13: Responsive Usability Across Mobile and Desktop**
  * Essential workflows (view dashboard, record payment, print receipt, check unit status) are executed on both a desktop browser (1920×1080) and a mobile smartphone viewport (375×667).
  * Zero horizontal overflow, broken components, or touch-target failures.

