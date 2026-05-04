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
  - User Subscription: Individual RM100/yr, Corporate RM200/yr (unlimited signing)
  - Transaction: Starter(500tx/RM5k) → Bronze(1k/RM8k) → Silver(3k/RM18k) → Gold(10k/RM40k) → Platinum(30k/RM60k) → Diamond(80k/RM80k)
- **Auth**: Email + Password
- **Security**: Private keys in FIPS 140-2 Level 3 HSM
- **Integrations**: DocuSign, Adobe Sign, Signing Hub, eIDeasy
- **Signer Onboarding**: Via MyTrustID app (NOT this portal) — Individual: MyDigital ID or eKYC; Corporate: LoA + SSM/LEI docs

## Roadmap
- **Q2 Phase 1** ← CURRENT: User management, Quota/usage monitoring, Reports
- **Q4 Phase 2** (TBD): Purchase plans, Subscription management, Payment integration

## Current Status
- **Last Session**: 2026-04-28 - Admin activation badge + Quota module planning
- **Next Steps**: Build Quota/Usage module — Plan + Subscription + SigningTransaction tables, then QuotaController + views
- **Pending Decision**: Transaction counter — live COUNT(*) from signing_transactions (Option A) vs cached quota_used column on subscriptions (Option B). User to decide tonight.
- **Known Issues**:
  - Need to configure MAIL_* in .env for invitation emails to send
  - Signer "Add Signer" form built but needs removal — signers register via MyTrustID app, not portal
  - KTDataTable `setFilter` column filter — test in browser to confirm DOM-mode filtering works

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

## Quota Module Plan (ready to build tonight)
Foundation tables needed:
- `plans` — id, name, type (subscription|transaction), quota_limit (null=unlimited), price_myr, duration_months
- `subscriptions` — id, client_id, plan_id, starts_at, expires_at, quota_used, status (active|expired|cancelled)
- `signing_transactions` — id, client_id, signer_id, signed_at, document_ref

UI:
- AT view: all clients — plan name, quota used/total, expiry, status badge
- AC view: own company — plan card, quota progress bar, usage by signer

Pending decision on transaction counter: live COUNT(*) vs cached quota_used column.

## Decisions Made
- **Signer sync**: Option B — shared DB (MyTrustID and portal share same DB)
- **Signer module**: Read-only for now (monitoring + AT status updates only), held pending further work

## Session History (Last 5)

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

### 2026-04-27 - Signer Module + Flow Clarification
- **Changes**:
  - Built `SignerType` enum (Individual/Corporate with cert descriptions)
  - Built `SignerStatus` enum (Pending/Active/Suspended/Revoked/Expired with badge colors)
  - Created `signers` table migration + `Signer` model + `SignerFactory`
  - `SignerController`: index, create, store, show, edit, update, updateStatus (AT only)
  - 7 routes: index, show, edit, update, update-status, create (nested under client), store
  - Views: index, create, show (with AT status panel), edit
  - **Flow clarification**: Signers register via MyTrustID app, NOT this portal
- **Time Spent**: ~1.5 hours

### 2026-04-27 - Client Management + Admin Invitation Flow
- **Changes**: Switched login to email+password. Fixed layout CSS variables. Added footer. Built Profile page. Built `ClientController`, `ClientAdminController`, `InvitationController`. Client code CLT-0001 auto-generated. Primary AC (lowest ID) = only one who can add/remove admins (max 3). Admin invitation flow: token → email → set password → activate.
- **Time Spent**: ~3.5 hours

## Historical Summary
Project started 2026-04-23 with PDS study and system design. Over 8 sessions spanning 4 days: built auth layer (email+password, role enum, Metronic login page), multi-tenant structure (clients table, client_id FK on users), code quality pass (14 auth tests, MySQL test DB), full client management module (register/edit/suspend, admin invitation flow via email token, activation badge in admin table), signer module scaffold (held as monitoring-only, shared DB sync decided), client management test suite (41 tests total) + GitHub Actions CI/CD, KTDataTable upgrade on clients index. Quota module planned and ready to build.

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
**Last Updated**: 2026-04-28 (Session 8) | **Position**: #1/10 Active
