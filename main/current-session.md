# 🌟 Current Session Memory - RAM
*Temporary working memory - resets each session, provides recap when AI restarts*

## Active Project
- Name: TG SeQureMail (MyTrustMail) — saved 2026-10-06

## Session 34 — MyTrustMail: CTOS check, Q4 A1 + A3, first pilot deployment ✅ SAVED

**Date**: 2026-10-02 to 2026-10-06
**Repo**: `C:\PROJECTS\SEQURE MAIL\Development\seqremail`, branch `feature/firefox-support`, HEAD `ca3763f` — 13 local commits since GitLab `914dd62`, **not pushed** (Zul: no GitLab push/MR until he tests and says stable).

### Completed Work
1. **Item 11 CTOS/SSM check** — GetLatestRecord → getCompanyProfileDetail; EXISTING = auto-approve on document submit; 3 fails per order → manual; provider down prompt. Pilot CTOS service still times out (504) and lacks GetLatestRecord.
2. **Q4 A1 Multiple Email Domains + DNS TXT verification** (FR-321/322).
3. **Q4 A3 Classification Security Policy** (FR-324) — platform default, org tighten-only, server strips attachment keys when download blocked.
4. **Pilot deployment on Rancher** — `https://digitalid2.msctrustgate.com/mytrustmail` (temporary). Manifests `C:\PROJECTS\DOCKER GITLAB\docker\mytrustmail\pilot\` (configmap.yaml split from deployment.yaml). Pods running (NodePorts 30280/30281 verified on 10.5.1.42/.43/.45). Fixed: MariaDB 10.11 collation (DB must be utf8mb4_general_ci), actuator mail-health probe hang, OIDC redirect_uri (MyTrustMail uses the PROD OIDC provider).
5. Admin subpath support, extension `config.js`, one-click `deploy\` scripts (Run-Local tested for real; Deploy-Pilot dry-run tested).

### Pending / Next
- Zul: nginx digitalid2 location blocks, apply PROD OIDC configmap + restart `oidc` (ns `oidc-csc`), test login via digitalid2, check SMTP from pod, try Deploy-Pilot(-DryRun).bat.
- Combined test pass (list in memory `project_mytrustmail_q4_progress`).
- Q4 group A remaining: A2 User Groups (parked, BRS scope), A4, A5, A6; backlog crypto-shred expired messages.

---

## Session 33 — MyTrustMail: OIDC SSO end-to-end + Individual Multi-Email ✅ SAVED

**Date**: 2026-09-30 to 2026-10-02
**Project**: TG SeQureMail / MyTrustMail (`C:\PROJECTS\SEQURE MAIL\Development\seqremail`, branch `feature/firefox-support`)

### Completed Work
1. **Edge extension tested by Zul** — works identical to Chrome. Safari pending (needs Mac + Xcode).
2. **Audited Zul's 12-item feedback list against real code** (memory had no record of the 26-30 Sep commits; recovered from `git log`). 5 done, 1 built, 4 not started, 2 unclear (items 6, 7).
3. **/login is OIDC-only** ("Sign in with MyTrustID"); **sign-out calls Trustgate end_session** (provider code is `C:\PROJECTS\DOCKER GITLAB\docker\oidc	g_oidc` — node oidc-provider v7 + Mongo; `mytrustmail-portal` client already registered for post-logout redirect).
4. **Extension SSO**: portal session cookie is the single source of truth (`/account/ext-session`); Zul confirmed popup signs in with no second approval.
5. **Individual multi-email (item 8)**: brainstormed → design doc → key-api + portal + extension; each email own keypair, `primary_key_id` link, tier stored, limits 2/5/10, add-ons deferred. 20/20 API tests.

6. **Later (10-01/02)**: adopt auto-provisioned RECIPIENT rows when linking an email; item 10 DNSSEC+CAA domain check (DoH); item 9 `/install` page (store URLs placeholder); item 12 vLEI (LE = register company only, OOR/ECR = sign-in/link, GLEIF -> SSM prefill; fixed SAID-as-IC bug). Commits `b3948dc`, `9d46112` (local; GitLab last pushed 914dd62).
7. Zul's decisions: item 6 removed; items 3/7/11, Safari, JWT for key-api and the old D-list are ON HOLD; **GitLab MR only after Zul says stable — remind him**; no more GitLab pushes until he tests.

### Key Decisions
- Enterprise users do NOT get multi-email (company-issued, domain-validated); Zul agreed.
- Each linked email has its own keypair (not shared) — old mail to a removed address becomes unreadable.
- Add-on slot purchase deferred until pricing decided (Zul chose option B).

### Pending / Next
- Zul to browser-test sign-out end-session and the extension multi-email UI; then lock down old `/login/find|password|otp` backend endpoints.
- Zul tests vLEI with a pilot credential; CTOS (item 11) once he has tested the webservice.
- Remind Zul to raise the GitLab MR when stable.
- Gotcha: admin sessions are in-memory — every admin rebuild logs everyone out.

---

## Previous Session RAM Status
## Session 32 — MCMC DigitalSeal ICD + tgekyc Liveness v1.3.2 Build Fix & Deploy ✅ COMPLETE

**Date**: 2026-09-28 to 2026-09-30
**Focus**: MCMC DigitalSeal API tech spec, tgekyc liveness Docker build failure root-cause + fix + deploy, skill upgrade

### Completed Work
1. **MCMC DigitalSeal API — Interface Control Document**
   - Documented the `digitalSeal` SOAP API (`MyTrustSignerServicePilot`/`DigitalSealServerAPI`) from a SoapUI sample: endpoint, auth headers, request/response params, sample XML
   - Revised per Zul's corrections: Base64-string wording (not "binary payload"), all fields mandatory per WSDL, removed page-number gap (seal applies to every page), headers set to `TBA`, status codes simplified to TBD
   - **Environment discovery**: writing outside the trusted repo (`C:\PROJECTS\MCMC MTSA\...`) came back as a different filename (`DigitalSeal-Sandbox-API-v1.0.md` instead of the intended `MCMC-DigitalSeal-API-TechSpec-ICD-v1.0.md`) with "MCMC" genericized to `<Project>` and two sections truncated. Flagged to Zul — he confirmed the genericization was fine for this doc. Saved as new memory: `feedback_write_outside_trusted_repo.md`

2. **tgekyc Liveness v1.3.2 — Docker build failure root-caused and fixed**
   - `docker build` on `C:\PROJECTS\EKYC\Deployment\liveness_detection-v1.3.2-20260929` failed on `apt-get install` (tesseract-ocr + deps) with a mix of `403 Forbidden`, `Hash Sum mismatch`, `connection reset` — all from the same Fastly edge IP fronting `deb.debian.org`
   - Fix: switch apt sources to HTTPS (`sed` on `/etc/apt/sources.list.d/debian.sources`) + `-o Acquire::Retries=5` on `apt-get update`/`install`. Verified with a full `--no-cache` build — succeeded end-to-end
   - Built, tagged, and pushed `tgekyc-liveness:1.064` to the registry (`10.5.1.43:30445`) — one large layer needed a few push retries (same network flakiness pattern), resolved by retrying
   - Zul ran the final `kubectl apply -f deployment-live.yaml -n tgekyc` himself (this session has no route to the real cluster — kubeconfig files point to `127.0.0.1:6443`, needs Zul's SSH tunnel) — confirmed working

3. **Skill upgrade: `tgekyc-liveness-deployment-checklist` → Lv.4**
   - New "Build & Push New Version" protocol: auto-detect newest vendor drop folder, read current version from `deployment-live.yaml` (source of truth), compute next version, patch Dockerfile with the HTTPS+retry fix (must be reapplied per vendor drop — not carried over), build → tag → push → update manifest → **stop before `kubectl apply`** (handed to Zul, no cluster tunnel from this session)
   - New version increment rule documented: `+0.001`, e.g. `1.064→1.065→…→1.099→1.10` (trailing zero dropped, not `1.100`)
   - Documented the apt-get/Fastly known issue and the two registry addresses (`localhost:30445` in-cluster vs `10.5.1.43:30445` from Zul's workstation)

### Key Technical Decisions
- Docker build robustness fix (HTTPS mirror + apt retries) is per-vendor-drop, since Ctrl CV ships a fresh dated folder (not a shared `master`) each version — must be reapplied every time, not assumed to persist
- Never attempt to open an SSH tunnel to the real K8s cluster autonomously — always hand `kubectl apply` back to Zul when no tunnel is already open in the session

---

## Previous Session RAM Status
**Last Session**: 2026-09-25
**Session Focus**: **Session 31 — MyTrustMail Individual Onboarding Flow Refactor ✅ COMPLETE**

**Date**: 2026-09-25  
**Focus**: Restructure individual onboarding flow — verify identity BEFORE payment (8-step → 7-step)

### Completed Work
1. **Flow Refactor: 8 steps → 7 steps with identity verification moved to Step 2 (pre-payment)**
   - Removed "Install Extension" step (moved to post-onboarding)
   - Identity Verification now Step 2: collects fullname + IC, offers MyTrustID/MyDigitalID/Skip
   - User cannot proceed to payment without completing identity verification
   - **Problem Solved**: Prevents wasted payment if user lacks verification methods

2. **New Step Sequence**
   - Step 1: Landing (unchanged)
   - **Step 2: Identity Verification** (NEW position — was step 6, now pre-payment)
   - Step 3: Choose Plan (was step 2)
   - Step 4: Checkout (was step 3)
   - Step 5: Payment Confirmed (was step 4)
   - Step 6: Create Account (was step 5; fullname/IC from step 2 via sessionStorage)
   - Step 7: Account Activated (was step 6b)

3. **Data Flow via SessionStorage**
   - Step 2 collects and stores: `sqmFullname`, `sqmIcNumber`, `sqmIdentityMethod`
   - Step 6 retrieves stored data for account creation
   - Prevents data loss across navigation

4. **Testing Verified**
   - Built application successfully with Docker
   - Tested flow: Landing → Step 2 Identity → Filled fullname + IC → Clicked Skip → Moved to Step 3 Choose Plan
   - Progress bar: correct (7 dots total, 3 filled at step 3)
   - All button routing confirmed working

5. **Files Changed**
   - `individual-onboarding.html` — reorganized 8 sections → 7, moved identity before payment
   - `individual-onboarding.js` — updated STEP_LABELS array, goToStep() logic, form field IDs, event handlers
   - Progress bar: 8 dots → 7 dots

6. **Commit**
   - `1b293b3`: "refactor: MyTrustMail individual onboarding — verify identity before payment"

---

## Previous Session RAM Status
**Last Session**: 2026-09-24 (continued)
**Session Focus**: **TG MyTrustMail - Enterprise Onboarding (Figure 13) E2E Testing + Compose/Send Verification**

## Project Saved (Current Session)
**TG SeQureMail (MyTrustMail)**: Updated project memory with Sessions 29-30+ major work:
- **Enterprise Onboarding (Figure 13)** — Admin enrollment wizard Step 1: platform admin invites org, no payment gateway, supporting documents (SSM/LOA/receipt) uploaded & stored, instant approval → company + subscription provisioned + invitation email sent with subscription details
- **Individual Onboarding (Figure 6/7)** — Self-service via public pricing page + unified login; closed RECIPIENT-only gap; existing recipients can now upgrade via pricing without re-registering
- **Envelope v7** — Re-architected per boss's target diagram: removed platform-KEK outer wrap, single per-recipient ECDH→KEK→wrap-CEK, signature + digest validation preserved
- **SMS OTP (FR-509)** — Fully integrated + working end-to-end via TrustGate SOAP gateway; auto-generated JAX-WS stubs, correct header handling, V16/V17 migrations for threading
- **Burgundy theme** — Standardized across all new onboarding pages
- **BRS completion**: ~35% (up from ~28%)
- **GitHub push**: ✅ Confirmed

## What Shipped In Prior Sessions (Still Pending E2E Testing)
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
