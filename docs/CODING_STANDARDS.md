# Engineering Standards & Coding Guidelines

**Document Version:** 2.0  
**Scope:** TypeScript, Node.js/Express Backend, React Frontend, Prisma Database Layer  

---

## 1. TypeScript Standards & Type Safety

1. **Strict Mode Enabled:** `strict: true`, `noImplicitAny: true`, and `strictNullChecks: true` must be enabled across all `tsconfig.json` files.
2. **Never Use `any`:** The use of `any` is strictly prohibited. Use `unknown` with runtime type narrowing or generic types when the type is not initially known.
3. **Explicit API Contracts:** Every service method, controller action, and utility function must declare explicit parameter and return types:
   ```typescript
   // BAD
   export const calculateDue = (charges, payments) => { ... }

   // GOOD
   export interface RunningBalanceResult {
     totalCharges: Prisma.Decimal;
     totalPayments: Prisma.Decimal;
     outstandingBalance: Prisma.Decimal;
   }

   export const calculateDue = (
     charges: Charge[],
     payments: PaymentAllocation[]
   ): RunningBalanceResult => { ... };
   ```
4. **Use Decimal for Currency:** Never cast database monetary values into JavaScript `number` for calculations. Always use `Prisma.Decimal` or `Decimal` from `decimal.js`.

---

## 2. Naming Conventions

* **Files & Directories:**
  * Kebab-case for general files, utilities, and directories: `date-utils.ts`, `auth-middleware.ts`, `water-billing/`.
  * PascalCase for React components: `TenantLedgerTable.tsx`, `RecordPaymentModal.tsx`.
* **Variables & Functions:** `camelCase`: `calculateBalance()`, `activeTenancy`, `periodKey`.
* **Types, Interfaces, Classes & Enums:** `PascalCase`: `PaymentAllocation`, `WaterSplitType`, `ChargeStatus`.
* **Constants & Environment Variables:** `SCREAMING_SNAKE_CASE`: `DEFAULT_CURRENCY`, `JWT_EXPIRES_IN`.
* **Database Tables & Fields (Prisma):** Model names in `PascalCase` (`Property`, `Unit`), fields in `camelCase` (`monthlyRent`, `archivedAt`).

---

## 3. Error Handling & Input Validation

1. **Server Validation with Zod:** Every incoming request body and query parameter must be validated through a Zod schema before hitting service logic:
   ```typescript
   export const CreatePaymentSchema = z.object({
     tenantId: z.string().cuid(),
     amount: z.string().regex(/^\d+(\.\d{1,2})?$/, "Invalid monetary format"),
     method: z.enum(["UPI", "BANK_TRANSFER", "CASH", "CHEQUE", "CARD", "OTHER"]),
     reference: z.string().optional(),
     paymentDate: z.string().datetime().optional()
   });
   ```
2. **Centralized Error Handling Middleware:** Never swallow errors with empty `catch` blocks. Controllers delegate errors to `next(err)`:
   ```typescript
   export class AppError extends Error {
     constructor(
       public statusCode: number,
       public code: string,
       message: string,
       public details?: unknown
     ) {
       super(message);
     }
   }
   ```
3. **HTTP Status Code Discipline:**
   * `200 OK`: Successful read or update.
   * `201 Created`: Successful creation.
   * `400 Bad Request`: Validation failure or business rule violation (e.g., negative payment).
   * `401 Unauthorized`: Missing or invalid authentication token.
   * `403 Forbidden`: Authenticated user does not own the requested resource.
   * `404 Not Found`: Resource does not exist (or belongs to another owner).
   * `409 Conflict`: Duplicate unique key (e.g. charge already issued for this month).
   * `500 Internal Server Error`: Unhandled technical exceptions.

---

## 4. Database Access & Transaction Discipline

1. **Transactional Boundaries:** Every operation modifying more than one database record must be executed inside `prisma.$transaction`:
   ```typescript
   // Atomic move-out settlement example
   await prisma.$transaction(async (tx) => {
     await tx.tenancy.update({
       where: { id: tenancyId },
       data: { status: 'ENDED', moveOutDate: new Date() }
     });
     await tx.unit.update({
       where: { id: unitId },
       data: { status: 'AVAILABLE' }
     });
   });
   ```
2. **Mandatory Owner Scoping:** Always verify ownership at the database query level:
   ```typescript
   // BAD: Vulnerable to Insecure Direct Object Reference (IDOR)
   const unit = await prisma.unit.findUnique({ where: { id: unitId } });

   // GOOD: Owner-scoped isolation
   const unit = await prisma.unit.findFirst({
     where: {
       id: unitId,
       property: { ownerId: authenticatedOwnerId }
     }
   });
   ```

---

## 5. Linting, Formatting & Prettier Standards

* **Prettier Configuration (`.prettierrc`):**
  * Single quotes: `true`
  * Trailing commas: `all`
  * Print width: `100`
  * Tab width: `2`
  * Semicolons: `true`
* **ESLint Configuration (`eslint.config.js`):**
  * Enforce `@typescript-eslint/recommended`.
  * Ban `console.log` in production code (use structured logging).
  * Enforce React Hooks dependency rules (`react-hooks/exhaustive-deps`).

---

## 6. Rules for Reviewing AI-Generated Code

When integrating code suggested or generated by AI assistants (including Copilot, ChatGPT, or Claude):
1. **Verify Isolation Boundaries:** Check that AI code did not introduce client-supplied `ownerId` trust or omit `where: { ownerId }` clauses.
2. **Check for Binary Floating Point:** Reject any AI code that uses `parseFloat()`, `Number()`, or basic math operators (`+`, `-`, `*`) on financial fields. Require `Prisma.Decimal` or integer math.
3. **Audit Error Handling:** Verify that the AI did not wrap operations in generic `try/catch` blocks that log and return `null`, which silently hides database failures.
4. **Inspect Generated Dependencies:** AI frequently proposes outdated or bloated npm packages. Reject any new third-party dependency if the task can be achieved cleanly with native Node.js or standard project libraries.
5. **No Hallucinated Schema Fields:** Verify that fields accessed by AI code actually exist in [`database/schema.prisma`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/database/schema.prisma).

---

## 7. Definition of Done (DoD) for Any PR

A pull request is ready to merge only when:
* [ ] Code compiles with zero TypeScript errors (`pnpm typecheck`).
* [ ] Linter and formatter pass without warnings (`pnpm lint`, `pnpm format:check`).
* [ ] Unit and integration tests cover new business logic and pass locally.
* [ ] Owner-scoped isolation has been explicitly verified with a cross-owner test case.
* [ ] Mobile responsiveness verified on small screen viewports (375px).
* [ ] Documentation (SRS, API specs) updated if domain rules changed.

