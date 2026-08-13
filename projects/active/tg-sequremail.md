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
- **Session 19 (2026-08-12)**: Project rebranded **SeQureMail → MyTrustMail** — pushed the `chore/rename-to-mytrustmail` branch (19 commits) after resolving a GCM unsafe-HTTP-remote push block. Read the full `BRS_MyTrustMail_V1.0.pdf` (13 functional modules, ~350 FR items) + 5 key workflow diagrams, saved a gap-analysis + roadmap reference doc to `docs/specs/2026-08-12-brs-gap-analysis.md`. Dejul set priority: Digital Signing module first (Chrome/Gmail scope), then admin portal polish. **Digital Signing module (envelope v5) implemented and verified this session** — see dedicated section below.
- **Session 18 (2026-08-10)**: Confirmed via `git status`/`git log` that `feature/9-recall-expiry-controls` is **already pushed** — up to date with origin, working tree clean. Corrects prior memory (session 17 recap said "not pushed"). Branch includes full `feature/6-user-management` history (branched off it), so an MR covering both is still open/needed.
- **Last Session**: 2026-07-21 (session 16) - Feature #9 (Recall & Expiry Controls) implemented on new branch `feature/9-recall-expiry-controls`, manually tested by Dejul in Chrome/Gmail — **confirmed working** after fixing a CORS bug found during testing (see below).
- **Next Steps**:
  1. ~~Manual Chrome/Gmail test of the new signing badge~~ — done 2026-08-13, confirmed working
  2. Admin portal polish — bring `mytrustmail-admin` up to BRS §5.2.3/§5.2.13 criteria (Dejul's stated next priority after signing)
  3. Open MR: `feature/9-recall-expiry-controls` → master (on GitLab) — covers Feature #6 + #9 together
  4. Open MR: `chore/rename-to-mytrustmail` → master
  5. Continue team feedback checklist: #5 (HSM), #4 (OWA), #2 (Firefox)
  6. **Parked (2026-08-13): CI/CD via Jenkins** — see dedicated section below, not started
- **Known Issues**: None

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

## Session History (Last 5)

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
- **Flyway Migrations (admin)**: V1 baseline | V2 (company_id + provisioned_by on user_keys, companies table) | V3 (admin_users) | V4 (drops legacy provisioned_by VARCHAR — superseded by key-api's V6) — isolated via `flyway_schema_history_admin`
- **provisioned_by_id**: self-referencing FK on `user_keys.id` — the sender (SUBSCRIBER) whose send-to-unregistered-recipient triggered auto-provisioning. Set in `CryptoServiceImpl.generateKeyPair(email, provisionedBy)`. NULL = admin pre-registered / self-registered (no triggering sender). Existing pre-fix rows stay NULL — no way to backfill who triggered them.
- **Fresh test protocol**: `TRUNCATE TABLE messages; TRUNCATE TABLE otp_verifications; TRUNCATE TABLE user_keys;` (in that order — no FK dependency between them) + `chrome.storage.local.clear()` in extension console + reload extension. DB access: `docker exec -it mytrustmail-db-1 mysql -umytrustmail -pmytrustmail123 mytrustmail_db` (container/creds renamed in the 2026-08-10 MyTrustMail rebrand — was `seqremail-db-1` / `seqremail` / `seqremail123` / `seqremail_db`)
- **Team Feedback Checklist** (branching strategy: one branch per feature, merge to master sequentially):
  - [x] #7 Non-subscriber role — `feature/7-non-subscriber-role` ✅ merged
  - [x] #6 User management — `feature/6-user-management` ✅ pushed, pending MR
  - [x] #9 Recall & Expiry Controls — implemented + verified 2026-07-21, on `feature/9-recall-expiry-controls` (pushed), pending MR. Sender-controlled only (no admin involvement). Envelope v4: double envelope moves server-side into new `messages` table, wrapped under a new Trustgate platform KEK (ECDH P-256) — email carries zero key material (`messageId`, `iv`, `ciphertext` only). Recall = crypto-shred (delete `wrapped_envelope` row) — kills decrypt for sender too, not just recipient. Expiry: per-user `default_expiry_days` default, overridable at compose time. New key-api endpoints: `POST /api/messages/{id}/recall`, `GET/PUT /api/keys/settings`; encrypt/decrypt updated with `MESSAGE_RECALLED`/`MESSAGE_EXPIRED` (HTTP 410). Migration `V7__recall_expiry.sql`. Extension: compose-time expiry picker, Recall button, terminal states. v3 (legacy) emails keep working, can't be recalled.
  - [x] **Digital Signing (not on the original numbered list — added from the 2026-08-12 BRS gap analysis, Dejul's #1 priority)** — implemented + API-verified 2026-08-12, envelope v5, `signing_public_key`/`signing_private_key` columns (V8). See dedicated section above for full detail. Chrome/Gmail manual UI test still pending.
  - [ ] #5 HSM integration
  - [ ] #4 Outlook Web (OWA) support
  - [ ] #2 Firefox support
  - [ ] #3 Safari support
  - [ ] #1 MyDigital ID onboarding
  - [ ] #8 Outlook plugin (if possible)
  - [ ] Admin portal polish to BRS §5.2.3/§5.2.13 criteria — Dejul's stated priority right after signing, not yet started

---
**Last Updated**: 2026-08-12 (session 19) | **Position**: #1/10 Active
