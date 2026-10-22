# Migrations, Seeding, and Database Operations

This guide provides the operational procedures for managing database migrations, test environments, and reproducible demo data for the **Property Management System (PMS)**.

---

## 1. Golden Rules of Database Management

> [!CAUTION]
> **CRITICAL PRODUCTION SAFEGUARD:**
> NEVER execute `prisma migrate reset`, `drop database`, or `prisma db push --force-reset` on a staging or production database. Running destructive commands on production will wipe financial histories and tenant data, violating audit compliance.

1. **Development Migrations Only:** Migrations must be generated and verified on local development databases before being committed.
2. **Review SQL DDL:** Always inspect generated SQL migration files (`migrations/YYYYMMDDHHMMSS_name/migration.sql`) before applying them to any shared environment.
3. **Automated Deployments:** In staging and production, only execute `prisma migrate deploy`, which applies pending migrations safely without schema resetting.

---

## 2. Local Development Database Setup

### 2.1 Prerequisites
* PostgreSQL 15+ installed locally or running via Docker.
* `pnpm` installed.

### 2.2 Docker Local Setup (Recommended)
You can start a lightweight PostgreSQL container locally using Docker:
```bash
docker run --name pms-postgres \
  -e POSTGRES_USER=pms_user \
  -e POSTGRES_PASSWORD=pms_password \
  -e POSTGRES_DB=pms_dev \
  -p 5432:5432 \
  -d postgres:16-alpine
```

### 2.3 Environment Variable
In your `.env` file:
```env
DATABASE_URL="postgresql://pms_user:pms_password@localhost:5432/pms_dev?schema=public"
DATABASE_TEST_URL="postgresql://pms_user:pms_password@localhost:5432/pms_test?schema=public"
```

---

## 3. Migration Workflow

### 3.1 Creating a New Migration
When making changes to [`database/schema.prisma`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/database/schema.prisma):
```bash
# Generate the migration and apply it to the local development database
pnpm db:migrate --name descriptive_migration_name
```
This will:
1. Compare `schema.prisma` with your local database schema.
2. Generate a versioned SQL file under `database/migrations/`.
3. Apply the SQL migration to `pms_dev`.
4. Regenerate the Prisma Client typescript bindings.

### 3.2 Inspecting Migration SQL
Check the generated `.sql` file to confirm:
* No unintended column drops (`DROP COLUMN`).
* Correct column types (e.g. `NUMERIC(12, 2)` for currency).
* Explicit foreign key constraints and `ON DELETE RESTRICT` clauses.

### 3.3 Safe Migration Deployment (Staging & Production)
```bash
pnpm prisma migrate deploy --schema=database/schema.prisma
```
This runs strictly the unapplied migrations recorded in `_prisma_migrations` table without prompting or resetting data.

---

## 4. Test Database Isolation

To prevent test runs from polluting or corrupting development data:
1. Always maintain a separate test database (`pms_test`).
2. Run automated tests with `NODE_ENV=test`, pointing `DATABASE_URL` to `DATABASE_TEST_URL`.
3. The test runner automatically truncates test tables between suites while preserving schema migrations.

---

## 5. Repeatable Demo Seeding

A comprehensive, deterministic seed script (`database/seed.js` or `database/seed.ts`) will be implemented in Phase 2 to populate a rich demo environment.

### 5.1 Demo Dataset Specification
The standard demo dataset contains:
* **1 Master Owner:** `owner@demo-pms.in` (Password: `DemoPMS@2026`).
* **3 Properties:**
  1. *Lotus Apartments* (Residential - 12 units).
  2. *Greenwood Villa* (Independent House - 1 unit).
  3. *Sunrise Commercial Plaza* (Commercial - 6 units).
* **15 Units:** Ranging from 1 BHK to 3 BHK and retail shops, rents from ₹8,000 to ₹45,000.
* **10 Tenants:** Complete with phone numbers, emails, and Aadhaar/PAN placeholders.
* **8 Active Tenancies:** With realistic move-in dates and security deposit statuses.
* **Billing & Ledger History:**
  - 3 months of historical rent charges (`2026-08`, `2026-09`, `2026-10`).
  - Realistic water bill meter readings and split charges.
  - Partial payments, fully paid charges, and 2 overdue tenant balances for realistic dashboard KPI demonstration.
* **Sample Documents:** Attached PDF placeholders for lease agreements and inspection reports.

### 5.2 Executing Seed
```bash
pnpm db:seed
```
The seed script will be completely idempotent: if run repeatedly, it clears demo entities cleanly or skips existing records based on unique identifiers.

---

## 6. Disaster Recovery & Emergency Reset (Development Only)

If a developer's local schema gets corrupted or out of sync during experimentation:
```bash
# ONLY run this in local development!
pnpm prisma migrate reset --schema=database/schema.prisma
```
This drops the development database, reapplies all historical migrations from scratch, and runs the seed script.

