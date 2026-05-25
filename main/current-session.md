# 🌟 Current Session Memory - RAM
*Temporary working memory - resets each session, provides recap when AI restarts*

## Session RAM Status
**Current Session**: Active
**Last Activity**: 2026-05-24
**Session Focus**: MyTrustSignerXML-MITI — new project created, full source code studied

## Active Project
- **Name**: MyTrustSignerXML-MITI
- **Started**: 2026-05-24
- **Context**: Java servlet API (Tomcat + MySQL) that signs XML files using XMLDSig standard (Apache Santuario + PKCS12). 3 endpoints: /sign, /verify, /getcertinfo. Currently working at sandbox/pilot. Due: 2026-06-01 (production deploy).
- **Next Steps**: Setup production environment — mtsa.properties, prod P12 cert, credential.json, MySQL transactions table, WAR deploy, test

## Previous Active Project (carry-over)
- **Resumed**: 2026-05-22
- **Last Worked**: 2026-05-22
- **Context**: WPF .NET Framework 4.8 desktop app — Token/SoftCert login & signing. Currently 98% complete.
- **Recent Progress**: AutoUpdate overhaul ✓ — asInvoker manifest, wss.txt→C:\Trustgate\MyTrustID\, VerseCheck IExpress runas, WSSCheck silent+restart, Dispatcher.Invoke fix, App.IsRestarting mutex race fix — branch fix/autoupdate-elevation pushed ✅
- **Next Steps**: merge fix/autoupdate-elevation → master; apply LNA box to page_auth_jnlp.jsp; bpfk team adds `allow="local-network-access"` to outer iframe; deploy updated JSPs to prod; sign EXE+MSI with SafeNet token

## 💭 Session Recap (For AI Restart)

### RSS Self Service Portal (Active — #1)
**Stack**: Laravel 13 + PHP 8.3 + Blade + Tailwind v4 + MySQL + Pest
**Completion**: 60%
**Repo**: `C:\laragon\www\remote-signing-portal`

**Billing model finalized (session 2026-05-06):**
- Subscription model: per signer/yr (Individual RM100, Corporate RM200), unlimited signing, added by AC/AT in portal
- Transaction model: per company shared quota block, deducts 1 per signing, signers via MyTrustID app
- One company = one model only (not simultaneous)
- Integrating with existing `billing`/`billing_users`/`billing_txsign` tables (shared with signing backend)
- Drop portal-scaffolded `subscriptions` + `signing_transactions` tables
- Only change to existing DB: add `client_id` to `billing` table
- Join key: `signers.ic_passport = billing_users.id_no`
- AT creates `billing` row manually; `billing.used` = external signings only; `billing_txsign` = all signings

**Next Steps (in order):**
1. ✅ Keep `is_internal` on signers
2. ✅ Transaction tiers confirmed — Starter(500/RM5k) → Diamond(80k/RM80k); Subscription: Individual RM100/USD25, Corporate RM200/USD50
3. Drop scaffolded `subscriptions` + `signing_transactions` tables
4. Migration: add `client_id` to `billing`, add `billing_id` to `clients`
5. Create Eloquent models: `Billing`, `BillingUser`, `BillingTxSign`
6. Service Plans UI (AT): AT creates billing row, sets quota
7. Signer management: AC adds internal signer → writes to `signers` + `billing_users`
8. Quota view (AC): `billing.quota - billing.used`
9. Dashboard live data + Reports

**Known issues / pending config:**
- Mail: MAIL_* not configured in .env — invitation emails go to log driver
- KTDataTable setFilter column indices — browser test needed for signers/clients tables

### Carry-over

**Windows Server Housekeeping (Active — #1):**
- 4 log paths: wildfly log, MyTrustSignServer logs/rsc/logs/DigitalSeal/log
- 10-day retention, forfiles approach
- Pending: user to confirm save path for .bat file on server

**MyTrustID Desktop (Active — #1):**
- Chrome PNA iframe fix DONE (2026-05-14) ✓
- auth.html deployed to bin\Debug\ web root
- bpfk.jsp updated with iframe+postMessage + 5-param URL passing
- Next: bpfk.gov.my server team deploys updated JSP; include auth.html in installer

**jumio-proxy-integration (Archived):**
- Implementation done — prod deploy pending (5 env vars to configure)

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
