# 🌟 Current Session Memory - RAM
*Temporary working memory - resets each session, provides recap when AI restarts*

## Session RAM Status
**Current Session**: 2026-09-23
**Session Focus**: **MITI MyTrustSignerXML - UAT Test Suite Creation (11 Essential Test Cases)**

## What Shipped Today
**MITI MyTrustSignerXML - Comprehensive UAT Test Suite (Streamlined)**
- Created 11 essential test cases (removed: audit logging, auth enforcement, exclusive C14N per Zul's direction)
- **Signing Tests (6)**: TC-SIGN-001 to TC-SIGN-006 covering basic signing, large payloads, error handling, whitespace normalization, object ID handling, concurrent requests
- **Verification Tests (3)**: TC-VERIFY-001 to TC-VERIFY-003 validating signature verification, tamper detection (signature), tamper detection (content)
- **Certificate Test (1)**: TC-CERT-001 certificate information retrieval
- **Digest Test (1)**: TC-DIGEST-001 SHA-256 digest calculation verification
- **Format**: Each test includes Objective (sentence form), Description, Test Procedure, Test Data (base64 trimmed to 20 chars), Prerequisite checklist, Expected Result (no checkboxes), Execution Result (PASS/FAIL tracking), and Endpoint URLs
- **Files Created**:
  - `UAT_TEST_CASES_FINAL.md` (v2.0 - fully formatted, 11 tests, endpoint URLs included)
  - `UAT_TEST_CASES_DETAILED.md` (v1.0 - reference version, 18 tests)
  - `UATIntegrationTest.java` (runnable test code, 5 executable methods)
  - `UAT_TEST_MATRIX.md` (quick reference matrix)
  - `UAT_EXECUTION_CHECKLIST.md` (execution tracking)
- **Quality**: Descriptions in full sentences, clear Expected Results (prescriptive, no checkboxes), execution tracking tables (PASS/FAIL)
- **Location**: `C:\PROJECTS\MITI\Development\MyTrustSignerXML\`
- Ready for immediate UAT execution

## Previous Session (2026-09-22)
**Subscription & License Management completion + Public Pricing/Unified Login (BRS 5.2.2 + 4.2.2) — 7 features shipped**

## What Shipped Today (chronological)

**1. License Transfer** (BRS 5.2.2)
- `license_transfers` audit table, `LicenseTransfer` entity
- `UserService.transferLicense()` — revoke source, promote destination, same-company validation
- Admin endpoint `/admin/users/{id}/transfer`
- Fixed 3 pre-existing schema bugs discovered along the way (V14 FK to nonexistent table, ENUM vs VARCHAR mismatch, duplicate @Override on CryptoServiceImpl.encrypt())
- Commit: `701d869`

**2. Subscription Renewal** (BRS FR-223, FR-227)
- `renewal_date` column (V8), `Subscription.renew()` — extends end_date +1 year
- Admin UI: green "Renew (1 Year)" button
- Commit: `59c4c6e`

**3. Subscription Upgrade — tier system** (BRS mockup: 5 tiers)
- `SubscriptionTier` enum: BASIC/STANDARD/ADVANCED (individual, 1 user) + BUSINESS_ADVANCED/ENTERPRISE (org, unlimited)
- `tier` column (V9), `Subscription.upgrade()` — auto-adjusts license count
- Admin UI: tier dropdown + Upgrade button, collapsible advanced status/license editor
- Commits: `97fdb2d`, `03b3e29`

**4. Bulk User Import polish**
- Email regex validation, 5MB/5000-row limits, CSV template download endpoint
- (Base upsert/dedup/error-reporting logic already existed from earlier session — just hardened it)
- Commit: `9333a39`

**5. Public Landing + Pricing Page** (BRS 4.2.2 self-service entry point)
- Discovered mid-conversation: **zero subscriber-facing web presence existed** — only admin portal login. Zul confirmed BRS requires individual self-service (Section 4.2.2, FR-223).
- New `PublicController`: `/`, `/about`, `/contact`, `/pricing` (all public)
- Pricing page: exact 7-tier mockup (Free Access, 30-Day Trial, Basic/Standard⭐/Advanced, Business Advanced👍/Enterprise)
- Lead capture: `/pricing/subscribe` → `subscription_leads` table (V10) — no payment gateway yet, admin converts manually (Phase 2)
- Commit: `22eaa0d`

**6-7. Unified Login (Step 1 + Step 2)**
- Problem: AdminUser (password) and UserKey/subscriber (OTP-only) had completely separate, incompatible auth mechanisms
- Step 1: `/login` — email-only entry, routes by table lookup (admin_users → password flow w/ prefill; user_keys → OTP flow; neither → pricing). Commit `c450c1e`
- Step 2: Real OTP web login for subscribers — `KeyApiOtpClient` (admin→key-api internal REST call, reuses key-api's existing OTP service, zero duplicated logic), programmatic Spring Security auth (no password path exists for subscribers), `/account` dashboard (FR-223 view: plan/period/licenses/renewal for company subscribers; simple view for individuals — no Subscription entity exists for standalone individuals yet, flagged as a gap)
- **Real bug hit and root-caused**: `CurrentAdminModelAdvice` (global `@ControllerAdvice`) threw `UsernameNotFoundException` for subscriber sessions — that exception type IS-A `AuthenticationException`, so Spring Security's `ExceptionTranslationFilter` silently redirected *every page, including permitAll ones*, back to `/admin/login`. Traced via a temporary debug endpoint dumping raw session state. Fixed by gating the advice to `ROLE_ADMIN` only.
- Verified end-to-end both directions: subscriber OTP→/account (200) + blocked from /admin (403); admin password→/admin (200) + blocked from /account (403)
- Commit: `597045a`

## Key Technical Decisions
1. **SubscriptionLead as a staging table** — captures pricing-page interest without building payment gateway yet; admin conversion UI is the next gap (not built)
2. **Admin app calls key-api internally for OTP** (RestTemplate, `KEY_API_URL` env var) — avoids duplicating OTP send/verify logic across two Spring Boot apps sharing one DB
3. **Programmatic Spring Security auth** for subscribers (no formLogin, since no password exists) — required explicitly exposing `SecurityContextRepository` as a bean, not just poking session attributes directly (pre-6.0 pattern doesn't reliably work in Spring Security 6.x)
4. **Role-based route separation**: `/admin/**` → ROLE_ADMIN, `/account/**` → ROLE_SUBSCRIBER, explicit `.hasRole()` matchers (not just `.authenticated()`)

## BRS Quarter Status (as of today, Q3 ends 2026-09-30 — 9 days left at session time)
- **Q2 2026: 12/12 = 100% done** ✅
- **Q3 2026: 9/14 fully done (64%)** — 1 partial (HSM: software done, hardware not), 2 blocked externally (MyDigital ID / MyTrustID — waiting Zul's pilot key), **2 not started (Edge + Safari extensions — biggest Q3 risk)**

## Known Gaps (explicitly flagged, not yet built)
1. **Lead conversion UI** — admin has no page to view/act on `subscription_leads` rows yet
2. **Payment gateway** — no integration, leads captured manually
3. **Individual subscriber tier tracking** — `Subscription` entity is company-only; standalone individuals have no formal plan record
4. **Self-service upgrade** — `/account` links to `/pricing` but doesn't let a subscriber change their own tier in-place
5. **SMS OTP** — deferred; Zul will provide his company's existing WSDL SOAP webservice details later (not a new gateway — see project_sequremail_sms_otp memory)
6. **Edge / Safari browser extensions** — zero progress, Q3 deadline imminent

**8. Edge Browser Extension** (BRS 5.2.9)
- Audited all `chrome.*` API usage across the extension — only `chrome.downloads`, `chrome.runtime`, `chrome.storage.local`, all natively Edge-compatible (Chromium-based since 2020). **Zero code changes needed.**
- Updated SETUP.md with Edge install steps; fixed stale path references left over from before the repo restructure (`mytrustmail-poc-v1/`, `mytrustmail-key-api/` subfolder)
- Could not test in actual Edge myself — browser automation tooling here is Chrome-only (confirmed via `list_connected_browsers`). Zul still needs to do a 5-min manual smoke test.
- Commit: `ab2125c`

**9. Account Deactivation (FR-119) + Self-service Upgrade path (FR-120)**
- `SubscriberStatus` was missing `DEACTIVATED` — only 4 of 6 BRS-required lifecycle states existed. Added it (both key-api and admin split-entity copies).
- New `UserDeactivatedException` (key-api), blocked in all 4 places `CryptoServiceImpl` already blocked SUSPENDED (claimKeyPair, sender check, recipient check, decrypt gate)
- Admin UI: Deactivate button (separate from Suspend), Reactivate now handles both SUSPENDED and DEACTIVATED starting states
- Self-service: `POST /account/deactivate` — subscriber deactivates their own account, logs out immediately, audit-logged as `SELF_DEACTIVATE`. Cannot self-reactivate (admin-only, same reasoning as SUSPENDED)
- **Bug caught mid-testing**: `SubscriberLoginController`'s OTP verify only checked SUSPENDED, not DEACTIVATED — a deactivated account could still complete login. Root cause: key-api's `verifyOtp()` only validates the OTP code, independent of `UserKey.status` (that gate lives in `claimKeyPair()`, a different call not used on repeat login). Fixed by adding the explicit check in the web login controller.
- FR-120: existing RECIPIENTs now reach `/pricing` via the same unified OTP login without re-registering — satisfies "no separate registration" requirement. Admin-side lead conversion remains unbuilt (flagged gap).
- Verified end-to-end: self-deactivate → DB updated → session killed → re-login blocked with correct message → dashboard access denied
- Commit: `e2325fe`

## BRS Deep-Dive: Module 5.2.1 (Subscriber Lifecycle Management) — FR-101 to FR-130
Went through all 30 FRs against actual code, not assumptions. Before today's FR-119/120 work: **9 Done / 9 Partial / 11 Gap**. Notable remaining gaps (not touched today): FR-101 (real self-registration via website — pricing page only captures a lead, doesn't create an account), FR-107/116 (auth method selection/management — moot until more than Email OTP exists), FR-114 (multi-email per account — hard unique constraint blocks this), FR-115/123 (admin password change/reset — no flow exists), FR-121 (Individual↔Enterprise migration), FR-127/128 (notification of lifecycle events, data retention — both tie to the still-missing Notification Services module, 5.2.8).

## Important Correction Made This Session
Initially conflated "gaps in the full 13-module BRS spec" with "gaps in the committed Q2/Q3 roadmap" — these are different. Modules 5.2.7 (Audit), 5.2.8 (Notification), 5.2.11 (Outlook), 5.2.12 (Email Classification) were **never on the committed Q2/Q3 roadmap** at all — building them was bonus work ahead of schedule, not catching up on a deadline. Corrected this framing explicitly with Zul mid-session. Worth remembering: always check the *committed* roadmap before flagging something as "behind schedule."

## Repo Status
- Branch: `feature/firefox-support`
- **Pushed to GitLab**: `e2325fe` (10 commits total this session, all pushed — one initial push hit a network/VPN hiccup, retried successfully after Zul reconnected)
- Docker: rebuilt many times during session, all 3 containers verified healthy at each step

## Next Steps (candidates, not yet decided)
1. Zul's manual Edge smoke test (5 min — load unpacked, one encrypt/decrypt cycle)
2. Lead conversion admin UI (quick win, ~1h — leads accumulating with no way to act on them)
3. SMS OTP (blocked — waiting Zul's WSDL details, see project_sequremail_sms_otp memory)
4. Payment gateway integration
5. Remaining 5.2.1 FR gaps (see deep-dive above)
6. Notification Services module (5.2.8) — ties into several FR gaps at once (FR-127, FR-227, FR-808, FR-627, FR-809)

---

**Timestamp**: 2026-09-22 completion (session ran across midnight)
**Total work**: Full-day session, 9 features + 1 architecture bug + 1 login-flow bug found/fixed, 10 commits pushed, one full BRS module (5.2.1) audited FR-by-FR against real code
