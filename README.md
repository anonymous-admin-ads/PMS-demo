# Property Management System (PMS)

[![CI Pipeline](https://github.com/organization/demo-pms/actions/workflows/ci.yml/badge.svg)](https://github.com/organization/demo-pms/actions/workflows/ci.yml)
[![Node Version](https://img.shields.io/badge/node-%3E%3D20.0.0-brightgreen.svg)](https://nodejs.org/)
[![Database](https://img.shields.io/badge/database-PostgreSQL%2015%2B-blue.svg)](https://www.postgresql.org/)
[![License](https://img.shields.io/badge/license-UNLICENSED-red.svg)](#)

A modern, lightweight, and auditable multi-property rental, billing, and accounting platform built for individual landlords and property owners.

---

## 1. Project Overview

The **Property Management System (PMS)** replaces paper rent books and spreadsheets with a single, high-performance web platform. It enables property owners to manage residential apartments, independent houses/villas, commercial spaces, and PG units (supporting 100+ units across a portfolio) across desktop, tablet, and mobile devices.

### Core Modules (10 Functional Domains):
1. **Executive Dashboard:** Live occupancy metrics, monthly billed vs collected revenue, and overdue arrears.
2. **Properties & Units:** Multi-property hierarchy, unit availability, asking rent, and security deposit specs.
3. **Tenants & Tenancies:** KYC profiles, non-conflicting active tenancies, and move-in/move-out workflows.
4. **Current Bills:** Consolidated monthly tenant statements combining rent, utilities, and maintenance.
5. **Charges:** Atomic debit line items with duplicate-billing prevention (`periodKey`).
6. **Water Bill:** Sub-metered water calculation and shared building utility split algorithms.
7. **Ledger:** Running statement of account with automated FIFO payment allocation and printable receipts.
8. **Reports:** Collection summaries, arrears aging, occupancy reports, and RFC 4180 CSV exports.
9. **Documents:** Secure, owner-scoped storage for lease agreements, ID proofs, and condition photos.
10. **Data Backup:** Automated database snapshots, SHA-256 verification, and disaster recovery procedures.

---

## 2. Selected Technology Stack

| Layer | Technology | Key Justification |
|---|---|---|
| **Frontend** | React 18 + Vite + TypeScript | Sub-150KB gzipped bundle; fast rendering on low-end mobile devices. |
| **Styling** | Tailwind CSS | Zero runtime CSS overhead, accessible touch ergonomics (44px targets). |
| **Backend** | Node.js (LTS 20) + Express + TypeScript | Lightweight REST API; minimal memory footprint (<80 MB RAM). |
| **Database** | PostgreSQL 15+ (Hosted on Supabase) | Managed cloud PostgreSQL with automated provisioning and transaction pooling (Supavisor). |
| **ORM** | Prisma ORM | Compile-time type safety, safe schema migrations (`directUrl`), and atomic transactions. |
| **Package Manager** | pnpm (Monorepo Workspaces) | Fast builds, strict dependency isolation, disk-efficient hard-linking. |

---

## 3. Distributed Documentation Index

All architectural guidelines, requirements, and operating procedures are maintained as living documentation:

| Document | Description |
|---|---|
| [`docs/SRS.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/SRS.md) | Master Software Requirements Specification, module scopes, and business rules. |
| [`docs/ARCHITECTURE.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/ARCHITECTURE.md) | System design, layering boundaries, performance principles, and technical rationale. |
| [`docs/API_AND_SCREENS.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/API_AND_SCREENS.md) | Screen layouts, REST API endpoints, request/response contracts, and status codes. |
| [`database/DATABASE_DESIGN.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/database/DATABASE_DESIGN.md) | Relational ER diagram, constraints, financial invariants, and indexing strategy. |
| [`database/MIGRATIONS_AND_SEEDING.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/database/MIGRATIONS_AND_SEEDING.md) | Safe migration practices, local Docker DB setup, and demo seed data rules. |
| [`docs/SECURITY.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/SECURITY.md) | Owner-scoped data isolation (anti-IDOR), token management, and threat modeling. |
| [`docs/TESTING_STRATEGY.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/TESTING_STRATEGY.md) | Testing pyramid, FIFO financial math tests, and test database isolation. |
| [`docs/DESIGN_SYSTEM.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/DESIGN_SYSTEM.md) | Mobile responsiveness, breakpoints, financial tables, and accessibility standards. |
| [`docs/DEPLOYMENT_AND_BACKUP.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/DEPLOYMENT_AND_BACKUP.md) | Low-cost hosting topology ($10–$17/mo), health checks, and backup restore protocol. |
| [`docs/GIT_AND_GITHUB_WORKFLOW.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/GIT_AND_GITHUB_WORKFLOW.md) | Conventional Commits, branch naming, PR checklist, and branch protection. |
| [`docs/OPEN_QUESTIONS.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/OPEN_QUESTIONS.md) | Architectural Decision Records (ADR), proration options, and client decision log. |
| [`planning/DEVELOPMENT_PLAN.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/planning/DEVELOPMENT_PLAN.md) | 9-phase implementation roadmap (Phase 0 through Phase 8). |
| [`planning/MILESTONES_AND_ACCEPTANCE.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/planning/MILESTONES_AND_ACCEPTANCE.md) | Measurable completion gates and the 13 master acceptance workflows. |
| [`AGENTS.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/AGENTS.md) | Constraints, rules against destructive commands, and instructions for AI agents. |

---

## 4. Getting Started & Development Setup

### 4.1 Prerequisites
* **Node.js:** `>= 20.0.0`
* **Package Manager:** `pnpm >= 9.0.0` (`npm install -g pnpm`)
* **Database:** Active Supabase project (or local Docker PostgreSQL for offline development)

### 4.2 Supabase Database Setup (Recommended)
1. Log in to [Supabase](https://supabase.com) and create a new project (e.g. `pms-dev`), selecting a region close to your target users (e.g., `ap-south-1` Mumbai).
2. Copy the environment configuration template:
   ```bash
   cp .env.example .env
   ```
3. From your Supabase project dashboard (**Project Settings $\rightarrow$ Database**):
   * Set `DATABASE_URL` to the **Connection Pooler** string (port `6543`, transaction mode with `?pgbouncer=true&sslmode=require`).
   * Set `DIRECT_URL` to the **Direct Connection** string (port `5432` with `?sslmode=require`).
4. Generate the Prisma Client:
   ```bash
   pnpm db:generate
   ```
5. Apply database migrations to Supabase:
   ```bash
   pnpm db:migrate
   ```

*(Optional Offline Development: Docker is supported for offline work without Supabase connectivity; see [`database/MIGRATIONS_AND_SEEDING.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/database/MIGRATIONS_AND_SEEDING.md) for details).*

### 4.3 Planned Development Commands
* `pnpm dev`: Starts both frontend (Vite) and backend (Express) in watch mode.
* `pnpm build`: Generates production builds for web and API.
* `pnpm typecheck`: Executes TypeScript compiler (`tsc --noEmit`) across workspaces.
* `pnpm lint`: Runs ESLint on all codebase files.
* `pnpm format:check`: Validates formatting via Prettier.
* `pnpm test`: Runs Vitest/Jest unit and integration test suites.
* `pnpm db:generate`: Regenerates Prisma client from `database/schema.prisma`.
* `pnpm db:migrate`: Executes pending migrations on the development database.
* `pnpm db:seed`: Seeds reproducible demo data.

---

## 5. Current Project Status

* **Active Phase:** **Phase 0 Complete** (Requirements baseline, distributed documentation, database design, and CI workflows established).
* **Next Immediate Phase:** **Phase 1: Repository Scaffolding & CI Activation** (Scaffolding `apps/web` and `apps/api` with strict TypeScript, ESLint, and Prettier configurations).
