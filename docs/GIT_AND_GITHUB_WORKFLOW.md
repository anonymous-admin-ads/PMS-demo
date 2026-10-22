# Git Standards, Conventional Commits & GitHub Quality Workflow

**Document Version:** 2.0  
**Scope:** Version Control, Branching, PR Lifecycle, CI Gates, Repository Protection  

---

## 1. Branching Strategy

The repository follows a streamlined **Trunk-Based / Feature-Branch Workflow**:
* **`main`**: Production-ready branch. Must always build, pass tests, and be deployable.
* **`develop`**: Integration branch for upcoming milestone releases (optional for small teams; features may branch directly from `main`).
* **Feature / Fix Branches:** Short-lived branches created for specific tasks or phases:
  * `feat/phase-2-auth-and-owner-scope`
  * `feat/phase-4-water-bill-calculator`
  * `fix/tenancy-move-out-status`
  * `docs/architecture-updates`
  * `test/fifo-allocation-edge-cases`

Branches should have a lifespan of no more than 2–3 days to prevent complex merge conflicts.

---

## 2. Conventional Commit Standards

Every commit message must follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:

```text
<type>(<scope>): <short imperative summary>

[optional detailed body explaining why the change was made]

[optional footer referencing issues, e.g. Closes #12]
```

### 2.1 Commit Types
* `feat`: A new feature for the user or API.
* `fix`: A bug fix in financial math, UI, or API routes.
* `docs`: Documentation updates only (e.g. `docs(srs): add water bill options`).
* `style`: Code style changes (formatting, whitespace; no functional logic change).
* `refactor`: Code reorganization without fixing bugs or adding features.
* `test`: Adding or correcting unit/integration tests.
* `chore`: Package updates, build configurations, or CI scripts.

### 2.2 Examples
```text
feat(billing): implement FIFO payment allocation for monthly rent charges

fix(tenancy): prevent overlapping active tenancies on the same unit

test(auth): add cross-owner IDOR security test cases for unit update
```

---

## 3. Pull Request (PR) Lifecycle & Reviews

1. **Keep PRs Small & Reviewable:** PRs should ideally touch fewer than 300 lines of code (excluding auto-generated lockfiles or schema diffs). Large features must be split across multiple incremental PRs.
2. **Use the PR Template:** Fill in the checklist in [`.github/PULL_REQUEST_TEMPLATE.md`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/.github/PULL_REQUEST_TEMPLATE.md).
3. **Mandatory Code Review Checklist:**
   * Is owner scoping enforced on all database queries?
   * Are financial fields using `Decimal(12, 2)` instead of binary numbers?
   * Are multi-entity mutations wrapped in `prisma.$transaction`?
   * Does the change include automated test coverage?
   * Is UI layout verified on mobile screens?

---

## 4. Local Quality Tooling & Git Hooks

### 4.1 Git Hooks (Husky & lint-staged - Planned for Phase 1)
To catch errors before code reaches GitHub:
* **Pre-commit:** Runs `lint-staged` to execute Prettier formatting and ESLint checks on staged files.
* **Commit-msg:** Validates that the commit message adheres to Conventional Commits format.

> [!NOTE]
> Git hooks are local developer convenience guards. Because local hooks can be bypassed (`--no-verify`), the automated GitHub Actions CI pipeline serves as the authoritative gatekeeper.

---

## 5. GitHub Actions Continuous Integration (CI)

The CI workflow defined in [`.github/workflows/ci.yml`](file:///c:/Users/skkin/OneDrive/Desktop/anonymous/demo-PMS/.github/workflows/ci.yml) triggers on all pushes and pull requests to `main` and `develop`.

CI enforces the following sequential checks:
1. **Repository Foundation Validation:** Verifies essential architecture documents and schema files are intact.
2. **Formatting Check:** `pnpm format:check` (Prettier).
3. **Static Linting:** `pnpm lint` (ESLint with TypeScript strict rules).
4. **Typechecking:** `pnpm typecheck` (`tsc --noEmit` across all workspace apps).
5. **Automated Testing:** `pnpm test` (Runs unit, integration, and security test suites).
6. **Production Build:** `pnpm build` (Verifies Vite and Express compile with zero errors).

---

## 6. GitHub Repository Settings & Branch Protection

> [!IMPORTANT]
> Branch protection is an administrative setting in the GitHub repository web interface. Creating a YAML file in `.github` does not automatically enforce branch rules until enabled in GitHub Repository Settings.

### Recommended GitHub Branch Protection Settings for `main`:
1. **Require a pull request before merging:**
   * Require at least 1 approving review.
   * Dismiss stale pull request approvals when new commits are pushed.
2. **Require status checks to pass before merging:**
   * `foundation-check`
   * `code-quality-and-tests`
   * Require branches to be up to date before merging.
3. **Require conversation resolution before merging:** All review comments must be marked resolved.
4. **Do not allow force pushes (`git push --force`)** and **Do not allow branch deletion**.

