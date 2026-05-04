# 🌟 Current Session Memory - RAM
*Temporary working memory - resets each session, provides recap when AI restarts*

## Session RAM Status
**Current Session**: Saved
**Last Activity**: 2026-04-29
**Session Focus**: RSS Self Service Portal — Quota module built, logos replaced, signer flow clarified

## Active Project
- **Name**: RSS Self Service Portal
- **Started**: 2026-04-23
- **Path**: `C:\laragon\www\remote-signing-portal`
- **Focus**: Next — Service Plans UI (AT) + Assign Subscription + Signer internal/external classification

## 💭 Session Recap (For AI Restart)

### RSS Self Service Portal (Active — #2)
**Stack**: Laravel 13 + PHP 8.3 + Blade + Tailwind v4 + MySQL + Pest
**Completion**: 60%
**Repo**: `C:\laragon\www\remote-signing-portal`

**What's done (session 2026-04-29):**
- Quota/Usage module fully built:
  - Migrations: `plans`, `subscriptions`, `signing_transactions`
  - Models: Plan, Subscription, SigningTransaction (with enums PlanType, SubscriptionStatus)
  - PlanSeeder: 8 plans seeded (Individual, Corporate, Starter→Diamond)
  - QuotaController: AT view (all clients summary) + AC view (own company detail)
  - Views: `quota/index.blade.php` (AT) + `quota/show.blade.php` (AC)
  - Transaction counter: live COUNT(*) — Option A (no cached column)
- Signers table made read-only in both `signers/index.blade.php` and `clients/show.blade.php`
- Signers/index upgraded to KTDataTable (search + status + type + sort filters, column-index-aware JS for AT/AC)
- Logos replaced: `logo-circle-2.jpeg` (sidebar, mobile header, footer) + `logo-login-2.jpeg` (login page)
- LARAVEL COMMAND.txt updated with 5 new artisan commands
- System design .txt updated to reflect current implementation

**Signer flow clarification (from team, 2026-04-29):**
- **Internal signers**: Added by AC/AT directly in portal → unlimited signing, NO quota deduction
- **External signers**: Registered via MyTrustID Mobile or MyDigitalID → quota IS deducted per signing
- Both types use MyTrustID Mobile or MyDigitalID to *approve* signatures
- Impact: need `is_internal` boolean on `signers` table, restore Add Signer form (for internal only), update quota COUNT to exclude internal signers
- Pending decision: boolean field vs enum for internal/external classification

**Next Steps (in order):**
1. Decide: `is_internal` boolean vs `SignerCategory` enum on signers table
2. Migration: add internal/external field to signers
3. Restore Add Signer form (for internal signers — AC and AT can add)
4. Update quota COUNT query to exclude internal signers
5. Service Plans UI (AT): `plans.index` route/controller/views
6. Assign Subscription to Client: AT assigns plan to client from client show page
7. Dashboard live data: wire up real DB counts
8. Reports: filter + CSV/PDF export

**Known issues / pending config:**
- Mail: MAIL_* not configured in .env — invitation emails go to log driver
- KTDataTable setFilter column indices — browser test needed for signers/clients tables

### Carry-over

**Windows Server Housekeeping (Active — #1):**
- 4 log paths: wildfly log, MyTrustSignServer logs/rsc/logs/DigitalSeal/log
- 10-day retention, forfiles approach
- Pending: user to confirm save path for .bat file on server

**MyTrustID Desktop (Active — #3):**
- Admin testing DONE (2026-04-21) ✓
- Next: Replace old `.bat` publish method with cleaner implementation

**jumio-proxy-integration (Archived):**
- Implementation done — prod deploy pending (5 env vars to configure)

**BIMB:**
- Waiting re-VAPT scheduling

## Key File Paths
- RSS Portal: `C:\laragon\www\remote-signing-portal`
- MTSA Dev: `C:\PROJECTS\MyGPKI-SKALA\Development\mtsa\`
- MTSA Deployment PILOT: `C:\PROJECTS\MyGPKI-SKALA\Deployment\PILOT\`
- MTSA Deployment PRODUCTION: `C:\PROJECTS\MyGPKI-SKALA\Deployment\PRODUCTION\`
- MyTrustID: `C:\repos\MyTrustIDv1_AATL-GENERIC\`
- Jumio Proxy: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\app\jumio-proxy\`
- switch-env: `C:\PROJECTS\MyGPKI-SKALA\Deployment\switch-env.ps1`
- Metronic template: `C:\PROJECTS\DOCUSIGN\Development\metronic-v9.4.9\metronic-tailwind-html-demos\dist\html\demo6\`

### Rules
- NEVER make code changes without Dejul's explicit permission first
- Always change development source files FIRST, then deployment copies
- MTSA JKR deployment: manually replace extracted .war → run `change env url .bat`

## 🔄 Auto-Reset Protocol
*Clear this file at start of next session after loading recap*
