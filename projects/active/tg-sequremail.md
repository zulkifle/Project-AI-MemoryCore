# TG SeQureMail - End-to-End Encrypted Gmail Extension
*Chrome extension POC for server-side HSM encrypted email via Gmail DOM interception*

## Project Overview
- **Type**: Chrome Extension (Manifest V3)
- **Client**: Any Gmail user (external-facing POC)
- **Period**: 2026-05-21 - Active
- **Tech Stack**: Frontend: Chrome Extension MV3 (thin client) | Backend: SeQureMail Key API (Spring Boot 3.2 + MySQL + Flyway) + seqremail-admin (Spring Boot 3.2, port 8081) — Software HSM | Crypto: ECDH P-256 + AES-256-GCM double-envelope (server-side, pure JDK) | Auth: OTP email verification (JavaMailSender)
- **Completion**: 99% (of original Phase 1 POC scope — see BRS gap analysis below for the much larger full-product scope)
- **Duration**: ~27 hours
- **Due Date**: TBD

## Current Status
- **Resumed**: 2026-08-12 (session 19) - from position #1 (never left top spot)
- **Session 22 (2026-08-24)**: Discovered (via direct `git log`/`git status` check — prior memory was stale) that sessions on 2026-08-19/20/21 had already fully finished and verified both Admin Multi-Tenant (26/26 test cases) and a brand-new **Subscriber Status Lifecycle** feature (Suspend/Reactivate, BRS §5.2.1 — not previously in memory at all), all committed locally but unpushed. **Pushed all 6 commits** to `chore/rename-to-mytrustmail` (`ce42335..f5d93a3`); confirmed via `git merge-base` that this branch already contains the full history of `feature/6-user-management` and `feature/9-recall-expiry-controls`, so **one MR covers all of it** — GitLab create-MR link: `http://gitlab.msctrustgate.com/trustgate/sequremail/-/merge_requests/new?merge_request%5Bsource_branch%5D=chore%2Frename-to-mytrustmail` (target `master`, not yet submitted). Then scoped **Firefox support** (BRS §5.2.9) via brainstorming skill — design approved, doc committed on new branch `feature/firefox-support` (off `chore/rename-to-mytrustmail` tip). See dedicated sections below.
- **Session 20 (2026-08-13→21, spans multiple actual work dates not reflected in earlier recaps)**: Admin multi-tenant designed + implemented + fully verified (26/26, 2026-08-19/20). Digital Signing module verified (2026-08-12). Subscriber Status Lifecycle (Suspend/Reactivate) designed + implemented + verified (10/10, 2026-08-20/21) — see dedicated sections below.
- **Session 19 (2026-08-12)**: Project rebranded **SeQureMail → MyTrustMail** — pushed the `chore/rename-to-mytrustmail` branch (19 commits) after resolving a GCM unsafe-HTTP-remote push block. Read the full `BRS_MyTrustMail_V1.0.pdf` (13 functional modules, ~350 FR items) + 5 key workflow diagrams, saved a gap-analysis + roadmap reference doc to `docs/specs/2026-08-12-brs-gap-analysis.md`. Dejul set priority: Digital Signing module first (Chrome/Gmail scope), then admin portal polish.
- **Next Steps**:
  1. Zul: submit the MR (`chore/rename-to-mytrustmail` → `master`) via the GitLab link above
  2. Implement Firefox support per the approved design (`docs/specs/2026-08-24-firefox-support-design.md`, branch `feature/firefox-support`) — manifest `browser_specific_settings.gecko.id` addition + manual test plan against real Firefox/Gmail; no JS logic changes expected
  3. Digital Signing — Chrome/Gmail manual UI test of the signature badge still not done (API-level only)
  4. Continue team feedback checklist: #5 (HSM), #4 (OWA)
  5. **Parked (2026-08-13): CI/CD via Jenkins** — see dedicated section below, not started
- **Known Issues**: None currently — working tree on `chore/rename-to-mytrustmail` was clean and fully pushed before branching for Firefox work. Local `feature/6-user-management` and `feature/9-recall-expiry-controls` branches are now redundant (fully contained in `chore/rename-to-mytrustmail`) — safe to delete after the MR merges.

## Parked — Jenkins CI/CD (2026-08-13, not started)
Discussed module-level BRS completion estimate (~19% of full BRS, much higher against Phase 1 scope only), then explored automating the API-level verification (encrypt/decrypt/signature/tamper/recall/expiry — the same checks done manually via direct API calls in sessions 16 & 19) as a functional/integration test, to eventually run in Jenkins. **Dejul chose to park this and return to feature dev first** — resume when ready.

**Findings so far**:
- Existing Jenkins: deployed on Rancher/K8s (`C:\PROJECTS\DOCKER GITLAB\docker\jenkins\deployment.yaml` + `deployment-agent.yaml`), namespace `jenkins`, web UI at `http://10.5.1.42:30450`, NodePort 30450 (http)/30451 (jnlp). This will be **the first pipeline** on this Jenkins instance — no existing Jenkinsfile to reference for credentials/pattern conventions.
- Current agent (`Agent1`) is a plain `jenkins/inbound-agent` pod — **no Docker access** (no socket mount, no DinD sidecar). `docker compose up` (needed to spin up db+key-api for the integration test) will not work on it as-is.
- Two options identified, not yet decided:
  1. Mount host Docker socket into the agent pod (Docker-outside-of-Docker) — simplest, but grants the pod host-root-equivalent access to the Rancher node; also unverified whether the node runs `dockerd` vs `containerd`-only.
  2. Register this Windows dev machine itself as a Jenkins agent (JNLP, same pattern as `Agent1` but on this host, not in K8s) — reuses the docker compose stack that already works here, avoids touching Rancher cluster security — **Jessy's recommendation**, lower risk, but ties CI to this machine being online.
- Planned pipeline scope (once resumed): Checkout → `mvn clean install` → `docker compose up -d` → run functional/integration test script (encrypt/decrypt/signature/tamper/recall/expiry, matching manual verification) → `docker compose down`. No deploy stage yet — build+test only.
- A PowerShell integration test script draft was written then reverted (encoding issues aside, Dejul wants to nail down the Jenkins agent architecture question first before investing in the test script).

## Feature #9 — Recall & Expiry Controls (implemented 2026-07-21, session 16)
Branch `feature/9-recall-expiry-controls` (off `feature/6-user-management`, includes its V6 migration). Full design at `docs/specs/2026-07-16-recall-expiry-controls-design.md`.

**Envelope v4**: content encrypted once under a random CEK (AES-256); CEK wrapped for recipient+sender (inner double envelope, same shape as v3) then wrapped again under a new **Trustgate platform KEK** (ECDH P-256, singleton row in new `platform_keys` table, generated lazily on first encrypt). Result (`wrapped_envelope`) stored server-side in new `messages` table — the email itself carries zero key material: `seqremail:poc-v4`, `from`, `to`, `messageId`, `iv`, `ciphertext` only.

**Recall** = crypto-shred: `POST /api/messages/{id}/recall` nulls `wrapped_envelope`. Sender-only (403 otherwise), idempotent, kills decrypt for the sender too (by design — same honesty as Outlook recall).

**Expiry**: `GET/PUT /api/keys/settings` manages `user_keys.default_expiry_days`. Resolution order: compose-time override > sender's default > never expires. Checked before content decrypt.

**New error codes**: `MESSAGE_RECALLED` / `MESSAGE_EXPIRED`, both HTTP 410 (Gone).

**Backward compat**: v3 (legacy, no `messageId`) envelopes keep decrypting via the unchanged old code path (`decryptV3Legacy` — byte-for-byte the pre-v4 logic, just extracted into its own method) — they simply can't be recalled.

**Refactor**: extracted `EcJwkUtil` + `AesGcmUtil` (new `util/` package) so `CryptoServiceImpl` and the new `PlatformKeyServiceImpl` share identical EC/JWK/AES-GCM primitives instead of duplicating ~80 lines.

**New files**: `entity/Message.java`, `entity/PlatformKey.java`, `repository/MessageRepository.java`, `repository/PlatformKeyRepository.java`, `service/PlatformKeyService.java` + impl, `controller/MessageController.java`, `dto/request/RecallRequest.java` + `UpdateSettingsRequest.java`, `exception/{ForbiddenException,MessageRecalledException,MessageExpiredException}.java`, `util/{EcJwkUtil,AesGcmUtil}.java`, migration `V7__recall_expiry.sql`.

**Extension**: compose modal gained an expiry picker (never/1/7/30 days, prefilled from sender's default via new `SeQureCrypto.getDefaultExpiry`); decrypted view gains a Recall button (confirm dialog, irreversible) when the viewer is the sender of a v4 message; recipient sees a dedicated terminal banner (no retry) on `MESSAGE_RECALLED`/`MESSAGE_EXPIRED`; popup gained a default-expiry select backed by `/api/keys/settings`.

**Verified end-to-end via direct API calls** (rebuilt Docker key-api container, migration V7 applied cleanly): v4 envelope shape confirmed zero key material; decrypt succeeds for recipient + sender, fails for a stranger; recall — 403 for non-sender, success for sender, idempotent on 2nd call, decrypt blocked for **both** parties afterward with `MESSAGE_RECALLED`; settings GET (null default) → PUT (7) → GET (7); encrypt without override correctly applied sender's stored default (checked `expires_at` in DB = `sent_at`+7d); 0-day expiry override correctly blocked decrypt with `MESSAGE_EXPIRED`.

**Manual Chrome/Gmail test — confirmed working (2026-07-21)**: Dejul reloaded the unpacked extension and exercised the real UI (expiry picker, send/decrypt, popup default-expiry setting). Hit one bug: PUT `/api/keys/settings` returned `403` — root cause was `CorsConfig.java` only allowlisting `GET, POST, OPTIONS`, so Spring's CORS filter rejected the PUT before it reached the controller. Fixed by adding `PUT` to `allowedMethods`, rebuilt + restarted the key-api container — confirmed resolved (404 for an unregistered test email instead of 403), then Dejul retried in the popup and confirmed success. Committed separately (`fix: allow PUT in CORS config...`) on top of the Feature #9 commit on `feature/9-recall-expiry-controls` — still not pushed.

**Fresh test protocol update**: gains `TRUNCATE TABLE messages;` (add to the existing `otp_verifications`/`user_keys` truncate, before `user_keys` — no FK dependency, just for cleanliness, per spec §7).

## Digital Signing Module — envelope v5 (implemented 2026-08-12, session 19)
Adds a second, separate ECDSA P-256 signing keypair per user (distinct from the existing ECDH P-256 encryption keypair) — satisfies the BRS's "Separate Signing and Encryption Key Management" requirement without waiting on real Trustgate CA/HSM integration (Dejul's explicit call: software keys now, matching Phase 1's actual architecture, upgradeable to real certs later).

**Flow**: `encrypt()` signs the plaintext with the sender's signing private key before encrypting (SHA256withECDSA), embeds the base64 signature in the envelope (`mytrustmail: poc-v5`). `decrypt()` verifies it with the sender's signing public key after decrypting, returning a structured result instead of a raw string: `{ plaintext, signed, signatureValid, signedBy }`. Envelopes without a `signature` field (pre-signing accounts / legacy v3-v4) get `signed:false` — not a verification failure.

**New files**: `util/EcdsaUtil.java` (sign/verify, SHA256withECDSA), migration `V8__add_signing_keys.sql` (idempotent, same conditional-DDL pattern as V6/V7), `dto/response/DecryptResponse.java`.

**Changed**: `UserKey` (signingPublicKey/signingPrivateKey columns), `CryptoServiceImpl.generateKeyPair()` (now generates both keypairs together), `CryptoServiceImpl.encrypt()`/`decrypt()`, `CryptoService` interface, `CryptoController.decrypt()` (returns `DecryptResponse` not raw string), extension `crypto.js` (`decryptMessage()` returns the full object) + `content_script.js` (`handleDecrypt()` renders a ✅/⚠️/ℹ️ signature badge in the success banner, pre-decrypt banner meta text now says "Signed (ECDSA P-256) + Encrypted (ECDH P-256 + AES-256-GCM)").

**Verified via direct API calls** (rebuilt Docker key-api container, migration V8 applied cleanly): full round-trip alice→bob confirmed `signed:true, signatureValid:true, signedBy:"alice@gmail.com"`; tampering the signature bytes (keeping ciphertext valid) correctly causes decrypt to fail rather than silently passing.

**Not yet done**: manual Chrome/Gmail test of the badge UI (API-level only so far — same two-step verification pattern used for Feature #9).

## Admin Multi-Tenant — Multiple Admins per Company (in progress, session 20, started 2026-08-13)
BRS §5.2.3/§5.2.13 slice, scoped via brainstorming skill — full design at `docs/specs/2026-08-13-admin-multi-tenant-design.md`. Invitation-based onboarding (email link, not a temp password) per BRS Figure 9/13. Dejul chose this slice over the alternatives (full Platform/Enterprise role split; platform dashboard) — deliberately kept to just: multiple `AdminUser`s per `Company`, invitation flow, and company-scoped data visibility.

**Done this session** (none of it compiled or run yet):
- Migration `V5__admin_multi_tenant.sql` (mytrustmail-admin) — `admin_users` gains `company_id` (nullable FK, null = platform-wide), `status` (PENDING/ACTIVE, backfills existing seeded admin to ACTIVE), `invitation_token`, `invitation_expires_at`; `password_hash` made nullable.
- `AdminUser` entity + new `AdminStatus` enum; `AdminUserRepository` gains `findByInvitationToken`/`findByCompanyIdOrderByCreatedAtAsc`.
- Mail capability added to mytrustmail-admin (`spring-boot-starter-mail`, same Gmail SMTP relay as key-api) + `ADMIN_BASE_URL` env var (docker-compose) for building the invite link.
- `AdminInviteService`/Impl — invite (creates PENDING admin + UUID token, 24h expiry, sends email), validateToken, acceptInvite (sets password, status=ACTIVE), remove.
- `InviteController` — public `GET/POST /admin/invite/accept?token=`, new `invite/accept.html` template; `login.html` updated with activated/pending messages.
- `SecurityConfig` — `/admin/invite/**` permitAll, custom `AuthenticationFailureHandler` mapping `DisabledException` → `?pending=true` (distinct message from wrong-password). `AdminUserDetailsService` — `disabled = status != ACTIVE`.
- `CurrentAdminService`/Impl — resolves the logged-in `AdminUser` from `Authentication` for scoping checks.
- `CompanyController` rewritten: invite/remove-admin endpoints (`POST /admin/companies/{id}/admins`, `.../admins/{adminId}/delete`); scoping — company-scoped admin's `/admin/companies` redirects to their own company, blocked from others, can't create/delete companies. `companies/detail.html` gained an Admins section (list + invite form + remove button).

**Done 2026-08-19 (tasks 9-11 of the 11-task plan — feature now fully implemented)**:
- `UserService`/`UserServiceImpl`/`UserController` — every user-facing method now takes a `forcedCompanyId` (null = platform admin unrestricted); list/ajax-search/add/CSV-upload/bulk role-change/bulk delete/single detail/single role/single delete all scope or reject accordingly. New `UserKeyRepository` count/findTop10 overloads scoped by `Company`.
- `DashboardController` — stats scoped by company; `totalCompanies` tile omitted entirely for scoped admins (`dashboard/index.html` grid drops 4→3 columns via `th:if`).
- `mvn compile` clean. Rebuilt full stack (`docker compose up --build`), migration V5 applied cleanly, no errors in logs.
- Ran the design doc's 5-step manual test plan (+ extra edge cases) via curl against the live containers: platform admin unrestricted ✅, invite→accept→login flow ✅, company-scoped dashboard/companies/users all correctly scoped ✅, cross-company reads/writes blocked (detail view, delete, company creation, company detail) ✅, admin removal revokes login ✅. All test data cleaned up afterward — DB back to pre-test state.
- Full test case list saved to `docs/specs/2026-08-19-admin-multi-tenant-test-cases.md` — 26 test cases total.

**Done 2026-08-20**: ran all 10 remaining code-only test cases via curl against the running containers (expired-token rejection, company-delete block, Add User form scoping, new-user/CSV-upload forced-company, own-company detail/role-change, bulk role/delete out-of-scope rejection, pending-invite cancellation) — all 10 passed on the first attempt, no code changes needed. Test data cleaned up. Commit `1ecb4c6`.

**Status: 26/26 test cases verified. Feature is sign-off ready.** Pushed to origin 2026-08-24 as part of `f5d93a3` (commit `f9287a6` for the feature itself).

## Subscriber Status Lifecycle — Suspend/Reactivate (implemented + verified 2026-08-20/21)
BRS §5.2.1 slice — design at `docs/specs/2026-08-20-subscriber-status-lifecycle-design.md`. Replaces the `claimed` boolean with a `SubscriberStatus` enum (Pending/Active/Suspended, plus an unused Expired placeholder for future licensing). Adds admin Suspend/Reactivate actions, company-scoped like existing role-change/delete. Dejul's call: no separate Deactivated status since hard-delete already covers permanent removal.

**Changed**: `encrypt()` now enforces status for both sender and an already-registered recipient (previously checked nothing); `decrypt()` distinguishes "not yet claimed" from "suspended" via new `USER_SUSPENDED` error (HTTP 403), same pattern as `MESSAGE_RECALLED`/`MESSAGE_EXPIRED`. `claimKeyPair()` refuses to reactivate a suspended account via OTP (closes an admin-lock bypass). Extension renders a terminal banner for `USER_SUSPENDED` the same way it does for message-level terminal states. New `V9__subscriber_status.sql` migration (key-api) drops `claimed`, adds `status`, backfills existing rows correctly.

**Verified**: 10/10 test cases via curl against rebuilt Docker stack (both containers rebuilt, `mvn compile` clean) — decrypt/encrypt blocking for suspended sender + recipient, reactivate restores access, fresh auto-provision unaffected (regression guard), OTP-claim defensive rule, dashboard status-count accuracy, company-scoped admin can suspend own-company user but not out-of-scope, existing delete action unaffected. Full list: `docs/specs/2026-08-20-subscriber-status-lifecycle-test-cases.md`. **Status: sign-off ready.** Commits: `e252939`/`fae19a2` (design), `f5d93a3` (implementation).

## Firefox Support (scoped 2026-08-24, not yet implemented)
BRS §5.2.9 slice — design at `docs/specs/2026-08-24-firefox-support-design.md`, branch `feature/firefox-support` (off `chore/rename-to-mytrustmail`). Scoped via brainstorming skill.

**Key finding**: extension's `chrome.*` API surface is tiny (5 call sites: `runtime.onInstalled`/`onMessage`/`sendMessage`, `storage.local`, `downloads.download`), all natively Firefox-compatible as written — **no JS logic changes needed**, only a manifest addition.

**Decisions**: single shared `manifest.json` for both browsers (add `browser_specific_settings.gecko.id`, Firefox-only field, harmless on Chrome) — no per-browser folder split; assume latest-stable Firefox only (older/ESR Firefox's lack of MV3 `service_worker` support accepted as a known, low-priority risk — fallback is swapping to `background.scripts` event page if it ever surfaces); distribution channel (AMO listed vs. unlisted self-distribute) deliberately deferred — doesn't block building/testing now, but note that Firefox requires Mozilla-signing to run in release Firefox even for internal-only distribution, so an AMO "unlisted" submission is the minimum eventual step; testing = manual test plan mirroring the existing fresh-test protocol, run against real Firefox + Gmail (register, encrypt/decrypt, signing badge, recall/expiry, suspend/reactivate, plus extra attention on `downloads.download` filename/extension behavior).

**Not yet done**: no code/manifest changes made yet — design only. Next: add `browser_specific_settings` to `manifest.json`, load unpacked in Firefox (`about:debugging`), run the manual test plan, document results in a dated test-cases doc.

## Session History (Last 5)

### 2026-08-13/17 (Session 20) - BRS Completion Estimate, Jenkins CI/CD Scoping (Parked), Admin Multi-Tenant (In Progress)
- **Changes**: Gave Dejul a module-level BRS completion estimate (~19% of the full 13-module BRS; much higher against Phase 1 scope alone) from the existing gap-analysis doc. Explored automating the API-level verification (encrypt/decrypt/signature/tamper/recall/expiry) as a Jenkins CI pipeline — found the existing Rancher-hosted Jenkins' only agent has no Docker access, identified two fix options, **parked at Dejul's request** to focus on feature dev instead (full detail in the dedicated Jenkins section above). Then scoped and designed "multiple admins per company" (BRS §5.2.3/§5.2.13) via the brainstorming skill — spec approved and implementation started (see dedicated section above for exact done/not-done breakdown). Session ended mid-implementation on "save project" — nothing built or tested yet.
- **Time Spent**: ~2.5 hours (estimate — no commit-based time tracking available, seqremail repo has no new commits this session)

### 2026-08-12 (Session 19) - MyTrustMail Rebrand Pushed, BRS Gap Analysis, Digital Signing Module
- **Changes**: Pushed the `chore/rename-to-mytrustmail` branch (19 commits, full SeQureMail→MyTrustMail rename) after committing the last pending piece (`docker-compose.yml` project name) and resolving a GCM unsafe-HTTP-remote push block. Read the full `BRS_MyTrustMail_V1.0.pdf` (13 functional modules, ~350 FR items) plus 5 key workflow diagrams, cross-checked against actual codebase entities/controllers, and saved a gap-analysis + roadmap reference doc to `docs/specs/2026-08-12-brs-gap-analysis.md`. Dejul set the priority order: Digital Signing module first (Chrome/Gmail scope only), then admin portal polish. Implemented and verified the Digital Signing module (envelope v5) — see dedicated section above.
- **Time Spent**: ~2.5 hours

### 2026-08-10 (Session 17) - Project Resumed
- **Changes**: Project resumed from position #6.
- **Time Spent**: ~0 min

### 2026-07-14 (Session 14) - Fresh Test E2E Confirmed + Feature #9 Proposed
- **Changes**: Resumed project. Ran full fresh-test E2E protocol (register both accounts, encrypt & send, decrypt on receiver side) — confirmed working end-to-end. Discussed and added **Feature #9: Revoke & Expiry Controls** to the team feedback checklist — corporate admin ability to instantly revoke access to a sent email or set an attachment expiration date, positioned as an enterprise selling point over native email. Feasible via a server-side gate (message-ID-keyed revoke/expiry flag checked before releasing decrypt material through the Key API), following the same pattern as the existing `claimed` flag gate. Not yet scoped or implemented.
- **Time Spent**: ~20 min

### 2026-07-07 (Session 13) - Admin Portal: Bulk Delete, Datatable UX, provisioned_by FK Fix
- **Changes**: Rebuilt + redeployed `seqremail-admin` container (Docker) with latest code. **Bulk delete**: `UserService.removeAll()` (batch delete), `POST /admin/users/delete-bulk` endpoint, "Delete" button added next to "Change" (renamed from "Change Role"/"Delete Selected") in the bulk action bar — reuses existing checkbox selection with its own confirm dialog. **Datatable UX**: converted Users table to fully client-side rendering (fetches via existing `/admin/users/ajax` endpoint on load, same as search) to support pagination (25/50/100/All per page, Prev/Next, "Showing X–Y of Z") and clickable sortable column headers (Company, Role, Status) with asc/desc arrow indicators — all state-managed in JS (`allUsers`, `sortField`, `pageSize`, `currentPage`). **provisioned_by fix**: discovered the column was always NULL — never wired up in code. Redesigned from `VARCHAR` (sender email, unused) to `provisioned_by_id BIGINT` self-referencing FK on `user_keys.id`, matching the `company_id` FK pattern. Updated both `UserKey` entities (key-api + admin) to `@ManyToOne` self-reference. Threaded sender's `UserKey` through `CryptoServiceImpl.encrypt()` → `resolvePublicKey()` → `generateKeyPair()` so future auto-provisioned recipients get `provisioned_by_id` populated. Migration split across two services due to compose startup order (key-api starts before admin): `V6` in key-api adds `provisioned_by_id` + FK (key-api's entity needs it at its own Hibernate validation); `V4` in admin drops the legacy `provisioned_by` string column. First attempt put the ADD COLUMN in admin's V4 — broke key-api startup (`SchemaManagementException: missing column`) since admin runs after key-api; fixed by moving the ADD to key-api's chain. Discussed (not implemented) future direction: storing X.509 certs per user + possible PDF/CMS signing — parked as a bigger scope conversation, closer to what MTSA already does.
- **Time Spent**: ~1.5 hours

### 2026-07-03 (Session 12) - Feature #6 Admin Portal Fully Implemented
- **Changes**: Scaffolded `seqremail-admin` Spring Boot 3.2 app (port 8081, branch `feature/6-user-management`). Full implementation: Spring Security form login (admin/BCrypt(7ru57ga73)), Thymeleaf 3 + Tailwind CDN (Swiss aesthetic — white, #E4002B accent, Inter font). Companies + Users + Dashboard pages. Flyway admin-isolated migrations (separate `flyway_schema_history_admin` table): V1 baseline, V2 add company_id + provisioned_by to user_keys (PREPARE/EXECUTE idempotent DDL for MySQL 8.4), V3 create admin_users. `FlywayConfig` auto-repair on startup. Dockerized: multi-stage build → root-level `docker-compose.yml` (db → key-api healthcheck → admin). Fixed bugs: Flyway V2 failed migration (depends_on service_healthy), Thymeleaf `:: head` fragment ambiguity (moved th:fragment to `<head>` element), 404 on `/admin/login` (added AuthController). **Feature additions this session**: (1) AJAX search with filter dropdown (All / Email / Company, 300ms debounce, `GET /admin/users/ajax` JSON endpoint, in-place table row re-render, no page reload); (2) Bulk role-change bar hidden until rows are checked, shows selected count, confirmation dialog before submit; (3) Add User form — role removed (hidden input, always SUBSCRIBER); (4) `UserKeyRepository` searchByEmail + searchByCompany JPQL queries; (5) Pre-registration pattern (null keys, claimed=false, OTP whitelist gate blocks unregistered). Extension not changed — all Feature #6 work is server-side.
- **Time Spent**: ~3 hours

## Historical Summary
Project started 2026-05-21 as a Chrome MV3 POC to prove client-side email encryption via Gmail DOM interception. Over 18+ sessions spanning almost three months, evolved from shared passphrase (Level 0) → RSA-OAEP keypair (Level 1) → ECDH P-256 client-side → ECDH P-256 server-side HSM with full security model + attachment support → admin management portal → rebranded MyTrustMail → digital signing. Key milestones: (1) Full SDD written 2026-05-26; (2) SDD formalised 2026-06-11; (3) POC redesigned to Level 1 2026-06-15 — DOM-based, no OAuth2; (4) Key API backend live 2026-06-15; (5) ECDH P-256 migration 2026-06-18; (6) OTP email verification 2026-06-24; (7) Server-side HSM 2026-06-24; (8) Auto-provision receiver keypair 2026-06-24; (9) Double-envelope + claimed flag 2026-06-25 — full security model; (10) Attachment auto-fetch 2026-06-25; (11) Documentation full update 2026-06-25 — all 7 docs rewritten to POC v3, threat model T1–T10; (12) Registration footer + noreply email removed + friendly error UX 2026-06-28; (13) Fresh test protocol + DB timezone fix (MYT) 2026-06-25; (14) Feature #7 non-subscriber role 2026-07-01; (15) seqremail-admin portal designed + built 2026-07-01 to 2026-07-03 — companies, users, bulk role change, CSV upload, AJAX search; (16) Admin portal bulk delete + sortable/paginated datatable + provisioned_by FK redesign 2026-07-07; (17) Feature #9 Recall & Expiry Controls (envelope v4) 2026-07-21; (18) Full rebrand SeQureMail → MyTrustMail 2026-08-10, pushed 2026-08-12; (19) Digital Signing module (envelope v5) 2026-08-12 — first module built against the new BRS gap analysis.

## Technical Notes
- **Repository**: Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\extension\` | API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\key-api\` | Admin: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\mytrustmail-admin\` (folder + Java package renamed from `seqremail-admin` in the 2026-08-10 rebrand; the top-level checkout directory itself is still named `seqremail`)
- **BRS reference**: `C:\PROJECTS\SEQURE MAIL\Documentation\Others\BRS_MyTrustMail_V1.0.pdf` + figures in `...\Others\BRS-figure\`. Gap analysis + product roadmap saved at `docs/specs/2026-08-12-brs-gap-analysis.md` (repo-relative).
- **Docker**: Root-level `docker-compose.yml` at `C:\PROJECTS\SEQURE MAIL\Development\seqremail\` (Compose project name `mytrustmail`) — starts db + key-api (healthcheck) + admin. Admin on port 8081. key-api on 8080. MySQL on 3307.
- **Admin Creds**: `admin` / `7ru57ga73` (BCrypt stored in admin_users table, seeded via DataLoader)
- **Design Doc (POC)**: `C:\PROJECTS\SEQURE MAIL\Documentation\POC\seqremail-poc-design-v2.md` (v1.5) — predates the MyTrustMail rebrand, filename/content not yet updated
- **Key Dependencies**: Chrome Extension MV3, MyTrustMail Key API (Spring Boot + MySQL), JavaMailSender (SMTP), Web Crypto API (file-only)
- **Crypto Plan**:
  - ~~v1: RSA-OAEP-2048~~ | ~~v2: ECDH P-256 client-side~~ | ~~v3: server-side double-envelope~~ | ~~v4: recall/expiry via messages table~~ | **v5 (current): adds ECDSA P-256 digital signing, separate keypair from encryption** | PQC planned per BRS roadmap (Phase 2, Q4 2026)
- **Envelope Format v5**: `mytrustmail: poc-v5`, `algorithm`, `from`, `to`, `messageId`, `iv`, `ciphertext`, `signature` (base64, ECDSA P-256, optional — absent for pre-signing accounts). Legacy v3 (`seqremail: poc-v3`, no messageId) and v4 (`mytrustmail: poc-v4`, no signature) still decrypt.
- **Security Model**: Auto-provision (`claimed=false`) → receiver registers via OTP → `claimed=true` → decrypt unlocked. Each party decrypts with own private key via own block. Strangers blocked by `selectBlock()` check. Signature (when present) verified against the sender's separate signing keypair after decrypt.
- **Flyway Migrations (key-api)**: V1 (user_keys) | V2 (otp_verifications) | V3 (private_key) | V4 (claimed flag) | V5 (role) | V6 (provisioned_by_id self-referencing FK) | V7 (messages + platform_keys tables, default_expiry_days — Feature #9) | V8 (signing_public_key/signing_private_key columns — Digital Signing module)
- **Flyway Migrations (admin)**: V1 baseline | V2 (company_id + provisioned_by on user_keys, companies table) | V3 (admin_users) | V4 (drops legacy provisioned_by VARCHAR — superseded by key-api's V6) | V5 (admin multi-tenant — company_id/status/invitation_token/invitation_expires_at on admin_users, password_hash nullable — not yet run/verified) — isolated via `flyway_schema_history_admin`
- **provisioned_by_id**: self-referencing FK on `user_keys.id` — the sender (SUBSCRIBER) whose send-to-unregistered-recipient triggered auto-provisioning. Set in `CryptoServiceImpl.generateKeyPair(email, provisionedBy)`. NULL = admin pre-registered / self-registered (no triggering sender). Existing pre-fix rows stay NULL — no way to backfill who triggered them.
- **Fresh test protocol**: `TRUNCATE TABLE messages; TRUNCATE TABLE otp_verifications; TRUNCATE TABLE user_keys;` (in that order — no FK dependency between them) + `chrome.storage.local.clear()` in extension console + reload extension. DB access: `docker exec -it mytrustmail-db-1 mysql -umytrustmail -pmytrustmail123 mytrustmail_db` (container/creds renamed in the 2026-08-10 MyTrustMail rebrand — was `seqremail-db-1` / `seqremail` / `seqremail123` / `seqremail_db`)
- **Team Feedback Checklist** (branching strategy: one branch per feature, merge to master sequentially):
  - [x] #7 Non-subscriber role — `feature/7-non-subscriber-role` ✅ merged
  - [x] #6 User management — `feature/6-user-management` ✅ pushed, pending MR
  - [x] #9 Recall & Expiry Controls — implemented + verified 2026-07-21, on `feature/9-recall-expiry-controls` (pushed), pending MR. Sender-controlled only (no admin involvement). Envelope v4: double envelope moves server-side into new `messages` table, wrapped under a new Trustgate platform KEK (ECDH P-256) — email carries zero key material (`messageId`, `iv`, `ciphertext` only). Recall = crypto-shred (delete `wrapped_envelope` row) — kills decrypt for sender too, not just recipient. Expiry: per-user `default_expiry_days` default, overridable at compose time. New key-api endpoints: `POST /api/messages/{id}/recall`, `GET/PUT /api/keys/settings`; encrypt/decrypt updated with `MESSAGE_RECALLED`/`MESSAGE_EXPIRED` (HTTP 410). Migration `V7__recall_expiry.sql`. Extension: compose-time expiry picker, Recall button, terminal states. v3 (legacy) emails keep working, can't be recalled.
  - [x] **Digital Signing (not on the original numbered list — added from the 2026-08-12 BRS gap analysis, Dejul's #1 priority)** — implemented + API-verified 2026-08-12, envelope v5, `signing_public_key`/`signing_private_key` columns (V8). See dedicated section above for full detail. Chrome/Gmail manual UI test still pending.
  - [x] **Admin multi-tenant / multiple admins per company (from the 2026-08-12 BRS gap analysis)** — 26/26 test cases verified, sign-off ready. See dedicated section above.
  - [x] **Subscriber Status Lifecycle / Suspend & Reactivate (BRS §5.2.1, not on the original numbered list)** — 10/10 test cases verified 2026-08-20/21, sign-off ready. See dedicated section above.
  - [ ] #5 HSM integration
  - [ ] #4 Outlook Web (OWA) support
  - [ ] #2 Firefox support — **design approved 2026-08-24** (`docs/specs/2026-08-24-firefox-support-design.md`), implementation not started. See dedicated section above.
  - [ ] #3 Safari support
  - [ ] #1 MyDigital ID onboarding
  - [ ] #8 Outlook plugin (if possible)
  - [ ] ~~Admin portal polish to BRS §5.2.3/§5.2.13 criteria~~ — in progress, see "Admin multi-tenant" item above

---
**Last Updated**: 2026-08-24 (session 22) | **Position**: #1/10 Active
