# AI Coding Agent Guidelines & Repository Constraints

This repository hosts a multi-property **Property Management System (PMS)**. All AI agents, contributors, and automation tools must strictly adhere to the guidelines and boundaries established below.

---

## 1. Mandatory Reading Before Any Code Changes
Before proposing, editing, or generating any code, database schemas, or API endpoints, read the following core specification documents:
1. [`docs/SRS.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/SRS.md) — Master business requirements, 10 core modules, and business rules.
2. [`docs/ARCHITECTURE.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/ARCHITECTURE.md) — Lightweight stack rationale, layering boundaries, and performance constraints.
3. [`database/DATABASE_DESIGN.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/database/DATABASE_DESIGN.md) — Relational model, constraints, and financial invariants.
4. [`docs/SECURITY.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/SECURITY.md) — Owner-scoped authorization, token handling, and multi-tenant security.
5. [`docs/CODING_STANDARDS.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/CODING_STANDARDS.md) — TypeScript guidelines, error formats, and transaction rules.

---

## 2. Cardinal Architecture & Security Constraints
* **Owner-Scoped Data Isolation:** Every database read, write, update, and delete MUST strictly filter by the authenticated `ownerId`. Never trust client-supplied owner IDs in query parameters or request bodies.
* **Financial Precision:** NEVER use binary floating-point arithmetic (e.g. JavaScript standard `Number`) for monetary values. Use Prisma `Decimal` or `decimal.js` with exact rounding.
* **Transactional Integrity:** All multi-step financial or lifecycle operations (payment allocation, move-out settlement, bill batching) MUST run inside an atomic `prisma.$transaction`.
* **Zero Overlapping Occupancy:** A unit must never have more than one `ACTIVE` tenancy simultaneously.
* **Audit Trail Preservation:** Historical financial and tenancy records must never be hard-deleted or cascaded when properties or tenants are archived. Use soft-delete / `archivedAt` flags.
* **Prohibited Infrastructure:** Do NOT introduce microservices, message brokers (Kafka/RabbitMQ), Redis caching, Turborepo, or heavy UI frameworks unless explicitly justified and approved by the Software Architect.

---

## 3. Strict Rules Against Unapproved Actions
* **NO Destructive Production Operations:** NEVER run `prisma migrate reset`, `drop database`, or force migrations against non-development environments.
* **NO Scope Creep:** Strictly implement features assigned to the active phase. Do not implement out-of-scope features (e.g., third-party payment gateways, automated WhatsApp bots, complex ERP role-hierarchies) without client sign-off.
* **NO Faked Tests or Stubs:** Never bypass failing tests or write tests that assert `true === true` to fake quality gates.

---

## 4. Standard Development & Verification Commands
* Run Development Servers: `pnpm dev`
* Run Static Typechecking: `pnpm typecheck`
* Run Linting & Formatting Checks: `pnpm lint` and `pnpm format:check`
* Run Test Suite: `pnpm test`
* Generate Prisma Client: `pnpm db:generate`
* Run Development Migrations: `pnpm db:migrate`

---

## 5. Definition of Done for AI Agents
1. All newly introduced code complies with [`docs/CODING_STANDARDS.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/docs/CODING_STANDARDS.md).
2. Automated unit or integration tests are added/updated for all business logic and financial edge cases.
3. Zero TypeScript compiler errors (`tsc --noEmit`).
4. Cross-owner security isolation is verified.
5. All UI components are verified for responsiveness on mobile (375px), tablet (768px), and desktop (1280px+).

