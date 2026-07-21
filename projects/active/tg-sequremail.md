# TG SeQureMail - End-to-End Encrypted Gmail Extension
*Chrome extension POC for server-side HSM encrypted email via Gmail DOM interception*

## Project Overview
- **Type**: Chrome Extension (Manifest V3)
- **Client**: Any Gmail user (external-facing POC)
- **Period**: 2026-05-21 - Active
- **Tech Stack**: Frontend: Chrome Extension MV3 (thin client) | Backend: SeQureMail Key API (Spring Boot 3.2 + MySQL + Flyway) + seqremail-admin (Spring Boot 3.2, port 8081) — Software HSM | Crypto: ECDH P-256 + AES-256-GCM double-envelope (server-side, pure JDK) | Auth: OTP email verification (JavaMailSender)
- **Completion**: 99%
- **Duration**: ~24.5 hours
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-07-21 (session 16) - Feature #9 (Recall & Expiry Controls) implemented on new branch `feature/9-recall-expiry-controls`, manually tested by Dejul in Chrome/Gmail — **confirmed working** after fixing a CORS bug found during testing (see below). Still committed locally, **not pushed** — Dejul hasn't said push yet.
- **Previous Session**: 2026-07-20 (session 15) - Resumed; git audit found + pushed unpushed session-13 code and docs commits (see Historical Summary)
- **Next Steps**:
  1. Push `feature/9-recall-expiry-controls` when Dejul gives the word, open MR
  2. Merge `feature/6-user-management` → master (MR on GitLab)
  3. Continue team feedback checklist: #5 (HSM), #4 (OWA), #2 (Firefox)
- **Known Issues**: None

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

## Session History (Last 5)

### 2026-07-14 (Session 14) - Fresh Test E2E Confirmed + Feature #9 Proposed
- **Changes**: Resumed project. Ran full fresh-test E2E protocol (register both accounts, encrypt & send, decrypt on receiver side) — confirmed working end-to-end. Discussed and added **Feature #9: Revoke & Expiry Controls** to the team feedback checklist — corporate admin ability to instantly revoke access to a sent email or set an attachment expiration date, positioned as an enterprise selling point over native email. Feasible via a server-side gate (message-ID-keyed revoke/expiry flag checked before releasing decrypt material through the Key API), following the same pattern as the existing `claimed` flag gate. Not yet scoped or implemented.
- **Time Spent**: ~20 min

### 2026-07-07 (Session 13) - Admin Portal: Bulk Delete, Datatable UX, provisioned_by FK Fix
- **Changes**: Rebuilt + redeployed `seqremail-admin` container (Docker) with latest code. **Bulk delete**: `UserService.removeAll()` (batch delete), `POST /admin/users/delete-bulk` endpoint, "Delete" button added next to "Change" (renamed from "Change Role"/"Delete Selected") in the bulk action bar — reuses existing checkbox selection with its own confirm dialog. **Datatable UX**: converted Users table to fully client-side rendering (fetches via existing `/admin/users/ajax` endpoint on load, same as search) to support pagination (25/50/100/All per page, Prev/Next, "Showing X–Y of Z") and clickable sortable column headers (Company, Role, Status) with asc/desc arrow indicators — all state-managed in JS (`allUsers`, `sortField`, `pageSize`, `currentPage`). **provisioned_by fix**: discovered the column was always NULL — never wired up in code. Redesigned from `VARCHAR` (sender email, unused) to `provisioned_by_id BIGINT` self-referencing FK on `user_keys.id`, matching the `company_id` FK pattern. Updated both `UserKey` entities (key-api + admin) to `@ManyToOne` self-reference. Threaded sender's `UserKey` through `CryptoServiceImpl.encrypt()` → `resolvePublicKey()` → `generateKeyPair()` so future auto-provisioned recipients get `provisioned_by_id` populated. Migration split across two services due to compose startup order (key-api starts before admin): `V6` in key-api adds `provisioned_by_id` + FK (key-api's entity needs it at its own Hibernate validation); `V4` in admin drops the legacy `provisioned_by` string column. First attempt put the ADD COLUMN in admin's V4 — broke key-api startup (`SchemaManagementException: missing column`) since admin runs after key-api; fixed by moving the ADD to key-api's chain. Discussed (not implemented) future direction: storing X.509 certs per user + possible PDF/CMS signing — parked as a bigger scope conversation, closer to what MTSA already does.
- **Time Spent**: ~1.5 hours

### 2026-07-03 (Session 12) - Feature #6 Admin Portal Fully Implemented
- **Changes**: Scaffolded `seqremail-admin` Spring Boot 3.2 app (port 8081, branch `feature/6-user-management`). Full implementation: Spring Security form login (admin/BCrypt(7ru57ga73)), Thymeleaf 3 + Tailwind CDN (Swiss aesthetic — white, #E4002B accent, Inter font). Companies + Users + Dashboard pages. Flyway admin-isolated migrations (separate `flyway_schema_history_admin` table): V1 baseline, V2 add company_id + provisioned_by to user_keys (PREPARE/EXECUTE idempotent DDL for MySQL 8.4), V3 create admin_users. `FlywayConfig` auto-repair on startup. Dockerized: multi-stage build → root-level `docker-compose.yml` (db → key-api healthcheck → admin). Fixed bugs: Flyway V2 failed migration (depends_on service_healthy), Thymeleaf `:: head` fragment ambiguity (moved th:fragment to `<head>` element), 404 on `/admin/login` (added AuthController). **Feature additions this session**: (1) AJAX search with filter dropdown (All / Email / Company, 300ms debounce, `GET /admin/users/ajax` JSON endpoint, in-place table row re-render, no page reload); (2) Bulk role-change bar hidden until rows are checked, shows selected count, confirmation dialog before submit; (3) Add User form — role removed (hidden input, always SUBSCRIBER); (4) `UserKeyRepository` searchByEmail + searchByCompany JPQL queries; (5) Pre-registration pattern (null keys, claimed=false, OTP whitelist gate blocks unregistered). Extension not changed — all Feature #6 work is server-side.
- **Time Spent**: ~3 hours

### 2026-07-01 (Session 11) - Feature #6 Admin Portal Design
- **Changes**: Brainstormed and designed `seqremail-admin` — separate Spring Boot app (port 8081, shared DB). Architecture: two-app model (key-api + admin), no REST calls between them, shared `seqremail_db`. Generated 5 Mermaid diagrams (`docs/admin-portal-diagrams.md`): system architecture, admin workflow, user registration flow, data model, OTP state diagram. Wrote full design spec (`docs/specs/2026-07-01-admin-portal-design.md`): companies table, admin_users table, V6 migration (company_id + provisioned_by on user_keys), CSV bulk upload, multiselect role change, user detail (OTP status + provisioned_by + platform). Registered `frontend-design` skill (v1.8.0). Key decisions: port configurable via env, no CSV row limit, billing = user count per company (external invoicing only, no payment gateway), single admin account (admin/BCrypt hash), whitelist-gated OTP.
- **Time Spent**: ~2 hours

### 2026-07-01 (Session 10) - Team Feedback + Feature #7 Non-Subscriber Role
- **Changes**: Received team feedback — 7-item feature checklist. Implemented **Feature #7 (Non-Subscriber Role)**: `UserRole` enum (SUBSCRIBER/RECIPIENT), V5 Flyway migration (`role VARCHAR(20) DEFAULT 'RECIPIENT'`), `GET /api/keys/role` endpoint, `assertSubscriber()` enforcement in `CryptoServiceImpl.encrypt()`, extension async role check greys out Encrypt toggle for RECIPIENTs. Branch `feature/7-non-subscriber-role` committed and pushed to GitLab. Explained MR vs PR (same concept, GitLab=MR, GitHub=PR). Explained sequential branch merging strategy for the 8-feature checklist.
- **Time Spent**: ~1 hour

## Historical Summary
Project started 2026-05-21 as a Chrome MV3 POC to prove client-side email encryption via Gmail DOM interception. Over 15+ sessions spanning six weeks, evolved from shared passphrase (Level 0) → RSA-OAEP keypair (Level 1) → ECDH P-256 client-side → ECDH P-256 server-side HSM with full security model + attachment support → seqremail-admin management portal. Key milestones: (1) Full SDD written 2026-05-26; (2) SDD formalised 2026-06-11; (3) POC redesigned to Level 1 2026-06-15 — DOM-based, no OAuth2; (4) Key API backend live 2026-06-15; (5) ECDH P-256 migration 2026-06-18; (6) OTP email verification 2026-06-24; (7) Server-side HSM 2026-06-24; (8) Auto-provision receiver keypair 2026-06-24; (9) Double-envelope + claimed flag 2026-06-25 — full security model; (10) Attachment auto-fetch 2026-06-25; (11) Documentation full update 2026-06-25 — all 7 docs rewritten to POC v3, threat model T1–T10; (12) Registration footer + noreply email removed + friendly error UX 2026-06-28; (13) Fresh test protocol + DB timezone fix (MYT) 2026-06-25; (14) Feature #7 non-subscriber role 2026-07-01; (15) seqremail-admin portal designed + built 2026-07-01 to 2026-07-03 — companies, users, bulk role change, CSV upload, AJAX search; (16) Admin portal bulk delete + sortable/paginated datatable + provisioned_by FK redesign 2026-07-07.

## Technical Notes
- **Repository**: Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\extension\` | API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\key-api\` | Admin: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\seqremail-admin\`
- **Docker**: Root-level `docker-compose.yml` at `C:\PROJECTS\SEQURE MAIL\Development\seqremail\` — starts db + key-api (healthcheck) + admin. Admin on port 8081. key-api on 8080. MySQL on 3307.
- **Admin Creds**: `admin` / `7ru57ga73` (BCrypt stored in admin_users table, seeded via DataLoader)
- **Design Doc (POC)**: `C:\PROJECTS\SEQURE MAIL\Documentation\POC\seqremail-poc-design-v2.md` (v1.5)
- **Key Dependencies**: Chrome Extension MV3, SeQureMail Key API (Spring Boot + MySQL), JavaMailSender (SMTP), Web Crypto API (file-only)
- **Crypto Plan**:
  - ~~v1: RSA-OAEP-2048~~ | ~~v2: ECDH P-256 client-side~~ | **v3 (current): SERVER-ECDH-P256-AES-256-GCM double-envelope** | v4 planned: ML-KEM-768
- **Envelope Format v3 (double)**: `seqremail: poc-v3`, `from`, `to`, `recipient: {ephemeralPublicKey, iv, ciphertext}`, `sender: {ephemeralPublicKey, iv, ciphertext}`
- **Security Model**: Auto-provision (`claimed=false`) → receiver registers via OTP → `claimed=true` → decrypt unlocked. Each party decrypts with own private key via own block. Strangers blocked by `selectBlock()` check.
- **Flyway Migrations (key-api)**: V1 (user_keys) | V2 (otp_verifications) | V3 (private_key) | V4 (claimed flag) | V5 (role) | V6 (provisioned_by_id self-referencing FK — added here, not admin, since key-api starts first in compose and its entity validates the column at startup)
- **Flyway Migrations (admin)**: V1 baseline | V2 (company_id + provisioned_by on user_keys, companies table) | V3 (admin_users) | V4 (drops legacy provisioned_by VARCHAR — superseded by key-api's V6) — isolated via `flyway_schema_history_admin`
- **provisioned_by_id**: self-referencing FK on `user_keys.id` — the sender (SUBSCRIBER) whose send-to-unregistered-recipient triggered auto-provisioning. Set in `CryptoServiceImpl.generateKeyPair(email, provisionedBy)`. NULL = admin pre-registered / self-registered (no triggering sender). Existing pre-fix rows stay NULL — no way to backfill who triggered them.
- **Fresh test protocol**: `TRUNCATE TABLE otp_verifications; TRUNCATE TABLE user_keys;` (in that order — no FK dependency between them) + `chrome.storage.local.clear()` in extension console + reload extension. DB access: `docker exec -it seqremail-db-1 mysql -useqremail -pseqremail123 seqremail_db`
- **Team Feedback Checklist** (branching strategy: one branch per feature, merge to master sequentially):
  - [x] #7 Non-subscriber role — `feature/7-non-subscriber-role` ✅ merged
  - [x] #6 User management — `feature/6-user-management` ✅ pushed, pending MR
  - [ ] #5 HSM integration
  - [ ] #4 Outlook Web (OWA) support
  - [ ] #2 Firefox support
  - [ ] #3 Safari support
  - [ ] #1 MyDigital ID onboarding
  - [ ] #8 Outlook plugin (if possible)
  - [ ] #9 Recall & Expiry Controls — **design approved 2026-07-16**, spec at `docs/specs/2026-07-16-recall-expiry-controls-design.md`, not yet implemented. Sender-controlled only (no admin involvement). Envelope v4: double envelope moves server-side into new `messages` table, wrapped under a new Trustgate platform KEK (ECDH P-256) — email carries zero key material (`messageId`, `iv`, ciphertext` only). Recall = crypto-shred (delete `wrapped_envelope` row) — kills decrypt for sender too, not just recipient. Expiry: per-user `default_expiry_days` default, overridable at compose time. New key-api endpoints: `POST /api/messages/{id}/recall`, `GET/PUT /api/keys/settings`; encrypt/decrypt updated with `MESSAGE_RECALLED`/`MESSAGE_EXPIRED` (HTTP 410). New migration `V7__recall_expiry.sql` (key-api only). Extension: compose-time expiry picker, Recall button injected into sender's own sent messages (confirm dialog, irreversible), new recipient terminal states. v3 (legacy) emails keep working, can't be recalled. Out of scope: admin portal, attachment-only expiry, un-recall, migrating old mail.

---
**Last Updated**: 2026-07-20 (session 15 — resumed) | **Position**: #1/10 Active
