ma
---

## Risk Matrix Summary

| ID | Risk | Likelihood | Impact | Level |
|---|---|---|---|---|
| R1 | Stack-auth Admin API rate limit during bulk import | High | Medium | 🟠 Medium |
| R2 | UUID preservation not supported by Stack-auth | Low | High | 🟠 Medium |
| R3 | Email duplicates across tenancies | Medium | Medium | 🟠 Medium |
| R4 | Google OAuth users cannot re-authenticate after migration | Low | High | 🟠 Medium |
| R5 | bcrypt hash direct import not supported by SDK | Low | Medium | 🟡 Low |
| R6 | `/api/stack-proxy` not added to `PUBLIC_PATHS` | Low | High | 🟠 Medium |
| R7 | PII leakage from SQL files in repository | Medium | Critical | 🔴 High |
| R8 | Migration interrupted mid-run | Medium | Low | 🟡 Low |
| R9 | `StackHandler` incompatibility with Astro | High | Medium | 🟠 Medium |
| R10 | Session cookie XSS exposure | Low | High | 🟠 Medium |
| R11 | Stack-auth SDK version breaking changes | Low | Medium | 🟡 Low |

---

## Detailed Risk Analysis

---

### R1 — Stack-auth Admin API Rate Limit During Bulk Import
**Likelihood:** High | **Impact:** Medium

**Description:**
The Stack-auth Admin API endpoint for user creation (`createUser` / `POST /api/v1/users`) enforces rate limits. At ~21,761 users, even modest per-second limits (e.g., 10 req/s) would cause 429 responses. Sending all requests concurrently would almost certainly trigger the limit.

**Mitigation:**
- **Primary:** Batch 50 users, process with `Promise.allSettled()`, add 200ms delay between batches. Estimated throughput: ~250 users/min → ~87 seconds total.
- **Secondary:** Exponential backoff on 429: 5s → 10s → 20s → abort. Checkpoint system resumes from last successful batch.
- **Tertiary:** Run migration during off-peak hours (Stack-auth rate limits may be per-project and shared with production traffic).
- **Pre-implementation:** Check Stack-auth Dashboard or docs for the exact admin API rate limit for the project's plan tier.

---

### R2 — UUID Preservation Not Supported
**Likelihood:** Low | **Impact:** High

**Description:**
The migration strategy preserves `projectUserId` as the Stack-auth user ID. This maintains referential integrity for any app data that references user IDs (reading history, bookmarks, etc. stored in Supabase).

If the Stack-auth `createUser()` API does not allow specifying a custom `id`, new UUIDs will be generated, breaking all foreign key references in the Supabase database.

**Mitigation:**
- **Pre-implementation action (blocking):** Inspect `@stackframe/stack` Server SDK source or API documentation to confirm whether `id` can be passed to `createUser()`.
- If UUID preservation is supported: proceed as planned.
- If not supported: design a UUID-mapping table in Supabase (`old_id → new_stack_id`) and update all foreign key references after migration. This significantly increases migration complexity.

---

### R3 — Email Duplicates Across Tenancies
**Likelihood:** Medium | **Impact:** Medium

**Description:**
`contacts.sql` contains rows from multiple tenancies. The same email (e.g., `khotruyenai@gmail.com`) may appear in more than one tenancy. If both rows slip through the filter, the second `createUser()` call will fail with a conflict error.

**Observed:** At least one admin email appears twice in `contacts.sql` under different `tenancyId` values.

**Mitigation:**
- Primary dedup gate: filter strictly by `mirroredProjectId = '23350c46-6d6a-48fc-bff3-f1c31009367a'` in Pass 2. Only users belonging to the production project are processed.
- Secondary: build a `Set<email>` during Pass 1. If an email appears multiple times, log it and keep only the `isPrimary=TRUE AND usedForAuth=TRUE` entry.
- Treat "user already exists" response from Stack-auth as `status: duplicate` (not an error) — log and continue.

---

### R4 — Google OAuth Users Cannot Re-authenticate After Migration
**Likelihood:** Low | **Impact:** High

**Description:**
~21,758 users are Google OAuth accounts (`passwordHash = null`). After migration, their accounts exist in Stack-auth but can only be authenticated via Google OAuth. If the Stack-auth project's Google OAuth provider is not configured before the migration goes live, these users will be locked out.

**Mitigation:**
- **Hard prerequisite:** Configure the Google OAuth provider in Stack-auth Dashboard **before** going live. Requires a Google Cloud OAuth 2.0 Client ID and Secret.
- Set `oauthProviders: ['google']` in the user payload during migration so Stack-auth links the account to Google OAuth correctly.
- After migration, verify that one Google OAuth user can complete the full sign-in flow (end-to-end smoke test).
- Communication plan: if the site switches to Stack-auth auth before users return, they see the "Đăng nhập bằng Google" button and can authenticate seamlessly.

---

### R5 — bcrypt Hash Direct Import Not Supported
**Likelihood:** Low | **Impact:** Medium

**Description:**
The migration plan assumes that `stackAdmin.createUser({ passwordHash: '$2b$10$...' })` passes the raw bcrypt hash directly, avoiding re-hashing and preserving existing passwords for the 2 email/password users.

If the Stack-auth SDK does not expose a `passwordHash` parameter and only accepts a plaintext `password`, the 2 email/password users cannot keep their existing passwords.

**Mitigation:**
- **Pre-implementation action:** Inspect `@stackframe/stack` SDK source to confirm the `createUser` signature.
- If hash import is supported: use it directly.
- **Fallback:** Create the 2 accounts with a random temp password, immediately trigger a password reset email. Users set a new password on first login. Impact: 2 users, zero data loss, minor UX friction.

---

### R6 — `/api/stack-proxy` Not Added to `PUBLIC_PATHS`
**Likelihood:** Low | **Impact:** High

**Description:**
`run-server.mjs` uses a `PUBLIC_PATHS` array to bypass the `private_access` cookie gate. If `/api/stack-proxy` is not in this list, all browser-to-Stack-auth API calls (login, register, token refresh) will be intercepted by the gate and receive a 302 redirect to `/tam-biet` instead of a JSON response.

This would silently break all auth flows when the `PRIVATE_PASSWORD` gate is active, with no obvious error to the user (the auth modal would appear to hang or return a generic error).

**Mitigation:**
- Add `/api/stack-proxy` to `PUBLIC_PATHS` in `run-server.mjs` as part of the implementation task.
- Add `/handler/` (OAuth callback path) to `PUBLIC_PATHS` as well.
- Write an integration test: with `PRIVATE_PASSWORD` set, a POST to `/api/stack-proxy/auth/v1/token` must return JSON (not a redirect).

---

### R7 — PII Leakage from SQL Files in Repository
**Likelihood:** Medium (was High before mitigation) | **Impact:** Critical

**Description:**
`database/users/contacts.sql` contains ~21,761 real email addresses. `database/users/project_users.sql` contains display names and Google profile image URLs. These are personal data (GDPR/privacy concern).

**Current status:** `database/users/` is already in `.gitignore` (fix applied in prior session). Files show as untracked — they will NOT be committed.

**Remaining mitigation:**
- Run `git status` before every commit during migration work to confirm `database/users/` remains untracked.
- After migration is complete and validated, **delete the SQL files from disk** (`database/users/*.sql`).
- Store any backup of the source dump in a secure, access-controlled location (not in this repo).
- Do not share or email the SQL files.

---

### R8 — Migration Interrupted Mid-run
**Likelihood:** Medium | **Impact:** Low

**Description:**
Network interruption, Stack-auth API timeout, or manual Ctrl+C mid-migration leaves the database in a partial state: some users migrated, others not.

**Mitigation:**
- Checkpoint system writes progress after every batch flush.
- `--resume` flag reads checkpoint and skips already-processed users.
- Partial migrations are safe: successfully inserted users can log in immediately; un-migrated users temporarily cannot, but no data is lost.
- Re-running without `--resume` skips duplicates gracefully (R3 dedup logic).

---

### R9 — `StackHandler` Incompatibility with Astro
**Likelihood:** High (confirmed) | **Impact:** Medium

**Description:**
The `StackHandler` component provided by `@stackframe/stack` uses Next.js-specific internals (`next/navigation`, `next/headers`, etc.) and cannot run in an Astro project. This affects all built-in Stack-auth handler pages (email verification, password reset, OAuth callback).

**Mitigation:**
- Do not use `StackHandler`. Write custom Astro pages for all required callback handlers:
  - `src/pages/handler/oauth-callback.astro` — Google OAuth return handler
  - `src/pages/handler/email-verification.astro` — email confirmation link handler (if email verification is enabled)
  - `src/pages/handler/password-reset.astro` — password reset link handler (for the 2 password users)
- Each custom handler calls the appropriate Stack-auth SDK method (e.g., `stackClientApp.handleOAuthCallback()`) then redirects.
- This is already accounted for in the `architecture.md` files-to-create list.

---

### R10 — Session Cookie XSS Exposure
**Likelihood:** Low | **Impact:** High

**Description:**
Stack-auth's `tokenStore: 'cookie'` stores the access token (`stack-access`) in a regular JavaScript-accessible cookie. Any XSS vulnerability in the site would allow an attacker to steal session tokens and impersonate users.

This is an inherent trade-off of the Stack-auth cookie tokenStore: it enables SSR session reading without an extra API call, but sacrifices HttpOnly protection.

**Mitigation:**
- Implement a strict Content-Security-Policy (CSP) header to prevent inline script injection.
- Audit all third-party scripts loaded on the page; minimize external JS dependencies.
- The access token has a short expiry (~1 day) — stolen tokens have a limited window.
- Consider enabling `Secure` and `SameSite=Strict` on cookies to reduce CSRF risk.
- If HttpOnly tokens become a requirement, evaluate Stack-auth's `tokenStore: 'memory'` option (session lost on page refresh) or a future HttpOnly-compatible store option.

---

### R11 — Stack-auth SDK Version Breaking Changes
**Likelihood:** Low | **Impact:** Medium

**Description:**
`@stackframe/stack` is an actively developed package. API changes between minor/major versions could break integration code after an `bun update`.

**Mitigation:**
- Pin the exact Stack-auth package version in `package.json` (use exact version, not `^` or `~`).
- Review Stack-auth changelog before updating.
- The integration is isolated in `src/lib/stack-client.ts` and `src/lib/stack-server.ts` — any API changes require updates only in these two files.

---

## Risk Register Summary

| ID | Risk | Owner | Status |
|---|---|---|---|
| R1 | Admin API rate limits | Dev | Batch 50 + 200ms delay + exponential backoff |
| R2 | UUID preservation | Dev | **Blocking pre-check:** verify SDK supports custom `id` |
| R3 | Email duplicates | Dev | Strict project filter + Set deduplication |
| R4 | Google OAuth not configured | DevOps | Configure Stack-auth Google provider before go-live |
| R5 | bcrypt hash import | Dev | Pre-check SDK; fallback = password reset for 2 users |
| R6 | Proxy not in PUBLIC_PATHS | Dev | Add `/api/stack-proxy` + `/handler/` to PUBLIC_PATHS |
| **R7** | **PII in SQL files** | **Dev** | **Done: `database/users/` in .gitignore. Delete after migration.** |
| R8 | Interrupted migration | Dev | Checkpoint + resume system |
| R9 | StackHandler incompatibility | Dev | Custom Astro handler pages |
| R10 | Session cookie XSS | Dev | CSP headers + minimal external JS |
| R11 | SDK version drift | Dev | Pin exact version in package.json |
