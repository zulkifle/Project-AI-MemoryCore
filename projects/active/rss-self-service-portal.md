# RSS Self Service Portal - Trustgate Remote Signing Services
*Self-service web portal for platform subscribers to manage remote signing users, monitor quota, and download reports*

## Project Overview
- **Type**: Web App (Portal)
- **Client**: Trustgate Sdn Bhd (internal product)
- **Period**: 2026-04-23 - Active
- **Tech Stack**: Laravel 13 + PHP 8.3 + Blade + Tailwind v4 + MySQL
- **Completion**: 60%
- **Duration**: 10 hours
- **Due Date**: Q2 2026 (Phase 1)

## Product Context (PDS)
- **Product**: Trustgate Remote Signing Services (RSS)
- **Certificate Types**: Individual Signer (AATL Basic, 1yr) | Corporate Signer (AATL Pro, 1-2yr)
- **Pricing Models**:
  - User Subscription: Individual RM100/USD25 per yr, Corporate RM200/USD50 per yr (unlimited signing)
  - Transaction tiers:
    | Plan | Block | USD | RM | Per Unit |
    |------|-------|-----|----|----------|
    | Starter | 500 | 1,250 | 5,000 | USD 2.50 / RM 10 |
    | Bronze | 1,000 | 2,000 | 8,000 | USD 2.00 / RM 8 |
    | Silver | 3,000 | 4,500 | 18,000 | USD 1.50 / RM 6 |
    | Gold | 10,000 | 10,000 | 40,000 | USD 1.00 / RM 4 |
    | Platinum | 30,000 | 15,000 | 60,000 | USD 0.50 / RM 2 |
    | Diamond | 80,000 | 20,000 | 80,000 | USD 0.25 / RM 1 |
- **Auth**: Email + Password
- **Security**: Private keys in FIPS 140-2 Level 3 HSM
- **Integrations**: DocuSign, Adobe Sign, Signing Hub, eIDeasy
- **Signer Onboarding**: Via MyTrustID app (NOT this portal) — Individual: MyDigital ID or eKYC; Corporate: LoA + SSM/LEI docs

## Roadmap
- **Q2 Phase 1** ← CURRENT: User management, Quota/usage monitoring, Reports
- **Q4 Phase 2** (TBD): Purchase plans, Subscription management, Payment integration

## Current Status
- **Last Session**: 2026-05-06 - Billing model + DB integration strategy finalized (discussion only)
- **Next Steps**:
  1. ✅ Keep `is_internal` on signers — retained for UI filtering
  2. ✅ Transaction plan tiers confirmed from PDS (see pricing table above)
  3. Drop scaffolded `subscriptions` + `signing_transactions` portal tables
  4. Migration: add `client_id` (nullable FK) to existing `billing` table
  5. Create Eloquent models: `Billing`, `BillingUser`, `BillingTxSign` mapped to existing tables
  6. Service Plans UI (AT): AT manages plan catalog only (not creating billing rows)
  7. Purchase Plan (AC): AC selects plan → FPX → system calls signing backend API → gets UUID → creates billing row
  8. Signer management: AC adds internal signer → writes to `signers` + `billing_users`
  9. Quota view (AC): reads `billing.quota - billing.used` per platform
  10. Dashboard live data + Reports
- **Known Issues**:
  - MAIL_* not configured in .env — invitation emails go to log driver
  - KTDataTable `setFilter` column indices — browser test needed for signers/clients tables

## Development Sequence (Phase 1)
- [x] Week 1: Auth (login/logout) + App layout + Dashboard skeleton
- [x] Client Management (AT): register, edit, suspend/reactivate, assign admins
- [x] Admin invitation flow: email invite → set password → activate account
- [x] Admin users table: show Active/Pending Invitation badge (filter by invitation_token)
- [~] Signer Management — scaffolded, held for now (monitoring-only, shared DB sync decided)
- [ ] Week 3: Quota / Usage display ← NEXT
- [ ] Week 4: Reports (filter + CSV/PDF export)
- [ ] Service Plans (AT): create plans, set quota
- [ ] Purchase Plan + FPX (AC)
- [ ] Buffer: Pest testing + polish + deployment

## Decisions Made
- **Signer sync**: Option B — shared DB (MyTrustID and portal share same DB)
- **Signer module**: Read-only for now (monitoring + AT status updates only), held pending further work
- **Billing integration**: Use existing `billing`/`billing_users`/`billing_txsign` tables — do NOT rebuild
- **Subscriptions table**: Drop portal-scaffolded version — `billing` IS the subscription record
- **Signing transactions table**: Drop portal-scaffolded version — `billing_txsign` IS the transaction log
- **billing_id on clients**: ❌ WRONG — dropped. Relationship is clients (one) → billing (many). Use `client_id` on `billing` instead.
- **One model per company**: ❌ WRONG — both subscription AND transaction can coexist under one `billing` row
- **AT creates billing manually**: ❌ WRONG — AC purchases plan → FPX payment → system auto-creates `billing` row via signing backend API
- **AT role on billing**: AT manages plan catalog only (plan tiers, pricing). Does NOT create billing rows.
- **AC role on billing**: AC purchases plan → triggers billing row creation automatically after payment
- **RSS scope rule**: RSS subscription cannot be used for other Trustgate products. But other platforms (DocuSign, Adobe Sign, Signing Hub, eIDeasy) CAN use RSS as signing engine.

## Existing Billing DB Schema (Source of Truth)
- **billing**: `id, name (DocuSign|UUID), description (company name), used, quota, date_added, date_modified`
  - `billing.name` = `{platform}|{UUID}` — platform = DocuSign/AdobeSign/etc, UUID = company ID in signing backend (generated by signing backend, not portal)
  - `billing.used` = external signing transactions consumed only (internal signers don't increment this)
  - `billing.quota` = transaction quota for external signers
  - One row per **platform per company** — a company using DocuSign + Adobe Sign = 2 billing rows
- **billing_users**: `id, billing_id, id_no (IC/MyKad), name, description, subscription_enddate, date_added, date_modified`
  - `subscription_enddate = null` = unlimited (no expiry)
  - `id_no` = natural join key to `signers.ic_passport` — no extra FK needed
  - Links internal (subscription) signers to a specific platform's billing row
- **billing_txsign**: `id, billing_id, type (1=Individual/2=Corporate), id_no, hash, signature, date_added, date_modified`
  - Logs ALL signings (internal + external)
  - Written by signing backend only — portal reads only

## Signing Logic
```
Signer signs via platform X → find billing row for company + platform X
  → check if id_no in billing_users WHERE billing_id = that row
      YES (+ valid subscription_enddate or null) → Internal → log to billing_txsign, skip quota deduction
      NO → External → log to billing_txsign + increment billing.used
```

## Purchase Flow (Phase 2 — FPX)
```
AC selects plan + platform → pays via FPX
  → payment confirmed
  → portal calls signing backend API → backend registers company → returns UUID
  → portal creates billing row: name = "{platform}|{UUID}", quota = plan quota
  → AC can now add internal signers (billing_users) or quota is live for external signers
```

## Session History (Last 5)

### 2026-05-06 (Session 14) - Billing Pricing Confirmed
- **Changes**: Discussion only. Confirmed: (1) keep `is_internal` on signers; (2) full transaction tier pricing from PDS locked in — Starter(500/RM5k) through Diamond(80k/RM80k) with USD equivalents; (3) subscription pricing confirmed Individual RM100/USD25, Corporate RM200/USD50 per year.
- **Time Spent**: ~15 min

### 2026-05-06 - Billing Model Corrections + DB Integration Finalized
- **Changes**: Discussion only — no code changes. Corrected several wrong assumptions from previous session. Key corrections: (1) both subscription + transaction can coexist under one billing row — not one model per company; (2) AT does NOT create billing rows — AC purchases plan → FPX → system auto-creates via signing backend API; (3) dropped `billing_id` on clients — use `client_id` on `billing` (one-to-many); (4) one `billing` row per platform per company — company using DocuSign + Adobe Sign = 2 rows; (5) RSS scope rule — RSS subscription locked to RSS only, but other platforms (DocuSign, Adobe Sign, Signing Hub, eIDeasy) can use RSS as signing engine; (6) UUID in `billing.name` generated by signing backend when company registers, not by portal.
- **Time Spent**: ~1.5 hours

### 2026-05-04 - Orientation Session
- **Changes**: No code changes. Confirmed project list path (`C:\PROJECTS\JESSY\Project-AI-MemoryCore\projects\project-list.md`), saved to Claude memory. Fixed save-memory skill incorrectly writing session history to relationship-memory.md — corrected to write to project file.
- **Time Spent**: ~10 min

### 2026-04-29 - Quota Module + Signer Flow Clarification
- **Changes**:
  - Migrations: `plans`, `subscriptions`, `signing_transactions` tables created
  - Models: Plan, Subscription, SigningTransaction (with PlanType, SubscriptionStatus enums)
  - PlanSeeder: 8 plans seeded (Individual/Corporate, Starter→Diamond tiers)
  - QuotaController: AT view (all clients summary) + AC view (own company detail)
  - Views: `quota/index.blade.php` (AT) + `quota/show.blade.php` (AC)
  - Transaction counter: live COUNT(*) — Option A chosen (no cached column)
  - Signers table made read-only in index + clients/show views
  - Signers/index upgraded to KTDataTable with search/status/type/sort filters
  - Logos replaced: `logo-circle-2.jpeg` + `logo-login-2.jpeg`
  - **Signer flow clarified**: Internal signers (added by AC/AT, unlimited signing, no quota deduction) vs External signers (via MyTrustID app, quota deducted per signing)
  - Pending: `is_internal` boolean vs `SignerCategory` enum decision
- **Time Spent**: ~3 hours

### 2026-04-28 - Admin Activation Badge + Quota Planning
- **Changes**:
  - `clients/show.blade.php` — Admin Users table: added Status column with Active (green) / Pending Invitation (yellow) badges. Slot count still counts both. Joined date shows dash for pending users.
  - Quota/Usage module planned: Plan + Subscription + SigningTransaction tables designed, UI mapped for AT (all clients summary) and AC (own company detail with progress bar)
  - Signer sync decided: Option B (shared DB). Signer module held as read-only.
- **Time Spent**: ~30 min

### 2026-04-28 - UI Improvement + Skill System Upgrade
- **Changes**:
  - `clients/index.blade.php` — full Metronic KTDataTable rewrite: card header with count + search + status/sort filters, company column member-style (initial + name + code), actions dropdown (`kt-menu-toggle`), footer with `data-kt-datatable-size` + `data-kt-datatable-pagination` (SVG prev/next), client-side mode via `KTDataTable.getInstance()` + `setFilter` + column header click for sort
  - `ClientController::index()` — simplified to `->get()`, removed server-side filter/sort/perpage logic
  - Jessy skill system: `laravel-php-skills` upgraded to Lv.2 (absorbed all 20 rules from deleted `laravel-best-practices`); `pest-testing` + `tailwindcss-development` trigger descriptions narrowed to avoid overlap
- **Time Spent**: ~2 hours

### 2026-04-27 - Client Management Tests + CI/CD
- **Changes**:
  - `tests/Feature/ClientTest.php` — 17 tests: access control (AT/AC/guest), create (code format, active default, name required), show/edit, suspend/reactivate, AC forbidden on status toggle
  - `tests/Feature/ClientAdminTest.php` — 10 tests: invite form access (AT, primary AC, non-primary blocked, wrong company), store invite (token generated, max 3 enforced, email unique), destroy (AT removes, primary AC removes, cannot remove self, wrong company blocked)
  - Total suite: 14 → **41 tests, 75 assertions**, all passing
  - `.github/workflows/tests.yml` — GitHub Actions CI: triggers on push/PR to main+develop, MySQL 8.0 service, PHP 8.3, Composer cache, migrate, `php artisan test --compact`
- **Time Spent**: ~45 min

## Historical Summary
Project started 2026-04-23 with PDS study and system design. Over 10 sessions spanning ~2 weeks: built auth layer (email+password, role enum, Metronic login page), multi-tenant structure (clients table, client_id FK on users), full client management module (register/edit/suspend, admin invitation flow via email token, activation badge), signer module scaffold (monitoring-only; signers register via MyTrustID app), client management test suite (41 tests, 75 assertions) + GitHub Actions CI/CD, KTDataTable on clients index, quota module (Plan/Subscription/SigningTransaction + QuotaController + views), signer internal/external flow clarified. Previous sessions also covered: signer type/status enums, 7 signer routes, AT status panel, profile page, layout CSS fixes.

## Technical Notes
- **Repository**: `C:\laragon\www\remote-signing-portal`
- **Key Dependencies**: Laravel 13, Tailwind v4, Pest v4, MySQL
- **DB Models**: User, Client, Signer, Plan, Subscription, SigningTransaction
- **Login**: email + password. Invitation flow for new admins.
- **User hierarchy**: AT (no client) → AC (client_id, max 3 per company)
- **Signer model**: Separate `signers` table — not Users. Fields: name, ic_passport, email, phone, signer_type, status, cert_serial, cert_expiry, notes
- **Primary AC**: lowest user ID among company ACs — only one who can add/remove other admins
- **Client status**: active | suspended
- **Signer status**: pending → active → expired/revoked/suspended
- **Plan Types**: subscription (unlimited/yr) | transaction (quota blocks)
- **Test accounts**: `at@rss.test` / password (AT) | `ac@rss.test` / password (AC, CLT-0001)
- **Dev Approach**: Step-by-step, one feature at a time
- **Testing**: MySQL test DB (`remote_signing_portal_test`). Use `->from('/url')` before POST for back() redirect assertions.
- **Mail**: Configure MAIL_* in .env (Mailtrap for dev) for invitation emails

---
**Last Updated**: 2026-05-06 (Session 14 — pricing confirmed) | **Position**: #1/10 Active
