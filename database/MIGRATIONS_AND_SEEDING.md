# Migrations, Seeding, and Database Operations (Supabase PostgreSQL)

This guide provides the operational procedures for provisioning a **Supabase PostgreSQL** project, managing Prisma schema migrations, configuring connection pooling, and seeding reproducible demo data for the **Property Management System (PMS)**.

---

## 1. Golden Rules of Database Management

> [!CAUTION]
> **CRITICAL PRODUCTION SAFEGUARD:**
> NEVER execute `prisma migrate reset`, `drop database`, or `prisma db push --force-reset` on a hosted Supabase staging or production database. Running destructive reset commands wipes historical ledgers, active leases, and audit trails.

1. **Keep `schema.prisma` as Source of Truth:** All table, column, enum, and index definitions must be authored in [`database/schema.prisma`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/database/schema.prisma).
2. **Review SQL DDL Before Committing:** Always inspect generated SQL migration files (`database/migrations/YYYYMMDDHHMMSS_name/migration.sql`) to verify that no unintended drops or data losses occur.
3. **Use Direct Connection for Migrations:** Supabase connection pooling (port 6543) does not support Prisma migration advisory locks or DDL transactions. Migrations must execute against the direct connection (`DIRECT_URL` on port 5432).
4. **Deploy Safely in Production:** In hosted environments, only execute `prisma migrate deploy`, which applies pending versioned migrations without resetting tables.

---

## 2. Supabase Project Setup & Connection Configuration

Follow these steps in the Supabase web console:

### 2.1 Project Provisioning
1. Log in to [Supabase](https://supabase.com) and click **New project**.
2. **Project Name:** `pms-dev` (or `pms-staging` / `pms-prod`).
3. **Database Password:** Generate a secure, high-entropy password (32+ characters). Store this securely in your password manager.
4. **Region Selection:** Choose a region geographically closest to your primary user base (e.g., **`ap-south-1` Mumbai** for India/INR deployments) to minimize round-trip query latency.
5. **Pricing Plan:** Select the **Free Tier** for development/testing, or **Pro Tier** for production (see [`docs/DEPLOYMENT_AND_BACKUP.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/DEPLOYMENT_AND_BACKUP.md) for plan trade-offs).
6. Click **Create new project** and wait for provisioning to finish (~1–2 minutes).

### 2.2 Obtaining Connection Strings
Navigate to **Project Settings** (gear icon) $\rightarrow$ **Database** $\rightarrow$ **Connection parameters**:

#### A. Direct Connection (`DIRECT_URL`)
* **Purpose:** Prisma CLI operations (`prisma migrate dev`, `prisma migrate deploy`, `prisma db push`).
* **Port:** `5432`
* **Format:**
  ```text
  postgresql://postgres:[YOUR-PASSWORD]@db.[PROJECT-REF].supabase.co:5432/postgres?sslmode=require
  ```

#### B. Connection Pooler (`DATABASE_URL`)
* **Purpose:** Express application runtime (queries handled by Prisma Client).
* **Port:** `6543` (Supavisor / Transaction Mode)
* **Parameters:** Append `?pgbouncer=true&sslmode=require` so Prisma Client configures prepared statement handling appropriately for transaction pooling.
* **Format:**
  ```text
  postgresql://postgres.[PROJECT-REF]:[YOUR-PASSWORD]@aws-0-[REGION].pooler.supabase.com:6543/postgres?pgbouncer=true&sslmode=require
  ```

---

## 3. Environment Variables Configuration

Copy `.env.example` to `.env` in the repository root and insert your Supabase credentials:

```env
# Runtime connection via Supabase connection pooler (port 6543)
DATABASE_URL="postgresql://postgres.yourprojectref:your_database_password@aws-0-ap-south-1.pooler.supabase.com:6543/postgres?pgbouncer=true&sslmode=require"

# Direct connection for Prisma CLI migrations (port 5432)
DIRECT_URL="postgresql://postgres:your_database_password@db.yourprojectref.supabase.co:5432/postgres?sslmode=require"

# Dedicated test database connection for automated test suites
DATABASE_TEST_URL="postgresql://postgres:your_database_password@db.yourprojectref.supabase.co:5432/pms_test?sslmode=require"
```

> [!TIP]
> **Optional Offline Development (Docker):**
> If developing offline without internet connectivity to Supabase, you can run a local PostgreSQL instance via Docker:
> ```bash
> docker run --name pms-postgres -e POSTGRES_USER=pms_user -e POSTGRES_PASSWORD=pms_password -e POSTGRES_DB=pms_dev -p 5432:5432 -d postgres:16-alpine
> ```
> In offline mode, set both `DATABASE_URL` and `DIRECT_URL` to `postgresql://pms_user:pms_password@localhost:5432/pms_dev?schema=public`. Docker is **not mandatory** when connecting directly to Supabase.

---

## 4. Prisma Migration Workflow

### 4.1 Step 1: Generate Prisma Client
Whenever `schema.prisma` is updated, regenerate the TypeScript types:
```bash
pnpm db:generate
# Underlying command: prisma generate --schema=database/schema.prisma
```

### 4.2 Step 2: Create and Apply Development Migrations
When adding new fields, models, or indexes in development:
```bash
pnpm db:migrate --name descriptive_migration_name
# Underlying command: prisma migrate dev --schema=database/schema.prisma
```
This command:
1. Connects to Supabase via `DIRECT_URL`.
2. Compares `schema.prisma` with the remote database state.
3. Generates a new migration folder under `database/migrations/YYYYMMDDHHMMSS_descriptive_migration_name/migration.sql`.
4. Executes the migration SQL against the database.
5. Regenerates the Prisma Client.

### 4.3 Step 3: Deploy Migrations to Staging & Production
In automated CI/CD pipelines and deployment runs:
```bash
pnpm prisma migrate deploy --schema=database/schema.prisma
```
This reads the migration history recorded in the `_prisma_migrations` table and applies only unexecuted migrations without prompting or resetting data.

---

## 5. Direct Connection vs. Connection Pooler (Technical Comparison)

| Dimension | Direct Connection (`DIRECT_URL`) | Connection Pooler (`DATABASE_URL`) |
|---|---|---|
| **Port** | `5432` | `6543` |
| **Technology** | Standard PostgreSQL Server | Supavisor (Supabase Next-Gen Pooler) |
| **Mode** | Session-based connection | Transaction-based pooling |
| **Connection Cap** | Limited by database compute (e.g. 15–60 connections) | Hundreds of concurrent client connections |
| **Prisma Migrations (`migrate dev/deploy`)** | **Required** (Supports DDL transactions & advisory locks) | **Incompatible** (Fails due to lack of advisory lock support) |
| **Prisma Client Runtime Queries** | Functional, but risks connection pool exhaustion | **Recommended** (Protects database from concurrent traffic spikes) |
| **SSL Enforcement** | Required (`sslmode=require`) | Required (`sslmode=require`) |

---

## 6. Repeatable Demo Seeding on Supabase

A deterministic seed script (`database/seed.js` or `database/seed.ts`) will be implemented in Phase 2 to populate the Supabase database with realistic property management data.

### 6.1 Demo Dataset Specification
* **Master Owner:** `owner@demo-pms.in` (Password: `DemoPMS@2026`).
* **3 Properties:**
  1. *Lotus Apartments* (Residential - 12 units).
  2. *Greenwood Villa* (Independent House - 1 unit).
  3. *Sunrise Commercial Plaza* (Commercial - 6 units).
* **15 Units:** Ranging from 1 BHK to 3 BHK and retail shops, rents from ₹8,000 to ₹45,000.
* **10 Tenants:** Complete with phone numbers, emails, and Aadhaar/PAN placeholders.
* **8 Active Tenancies:** With realistic move-in dates and security deposit tracking.
* **3 Months of Financial History:** Rent charges, water meter calculations, partial payments, and ledger balances.

### 6.2 Executing the Seed
```bash
pnpm db:seed
```
The seed script will be completely idempotent, checking for existing records before inserting to prevent duplicate entries on hosted Supabase environments.
