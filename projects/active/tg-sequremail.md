# TG SeQureMail - End-to-End Encrypted Gmail Extension
*Chrome extension POC for server-side HSM encrypted email via Gmail DOM interception*

## Project Overview
- **Type**: Chrome Extension (Manifest V3)
- **Client**: Any Gmail user (external-facing POC)
- **Period**: 2026-05-21 - Active
- **Tech Stack**: Frontend: Chrome Extension MV3 (thin client) | Backend: SeQureMail Key API (Spring Boot 3.2 + MySQL + Flyway) — Software HSM | Crypto: ECDH P-256 + AES-256-GCM double-envelope (server-side, pure JDK) | Auth: OTP email verification (JavaMailSender)
- **Completion**: 99%
- **Duration**: 17.5 hours
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-07-01 (session 10) - Team feedback received + Feature #7 (non-subscriber role) implemented
- **Next Steps**:
  1. Continue team feedback checklist — next feature TBD
  2. Full E2E test — register both accounts, encrypt & send, decrypt on receiver side
  3. Confirm attachment decrypt still works after fresh registration
- **Known Issues**: None

## Session History (Last 5)

### 2026-07-01 (Session 10) - Team Feedback + Feature #7 Non-Subscriber Role
- **Changes**: Received team feedback — 7-item feature checklist. Implemented **Feature #7 (Non-Subscriber Role)**: `UserRole` enum (SUBSCRIBER/RECIPIENT), V5 Flyway migration (`role VARCHAR(20) DEFAULT 'RECIPIENT'`), `GET /api/keys/role` endpoint, `assertSubscriber()` enforcement in `CryptoServiceImpl.encrypt()`, extension async role check greys out Encrypt toggle for RECIPIENTs. Branch `feature/7-non-subscriber-role` committed and pushed to GitLab. Explained MR vs PR (same concept, GitLab=MR, GitHub=PR). Explained sequential branch merging strategy for the 8-feature checklist.
- **Time Spent**: ~1 hour

### 2026-06-28 (Session 9) - Registration Footer + UX Fixes
- **Changes**: **Registration footer**: added plaintext footer below encrypted block in `content_script.js` — 4-step guide (install extension → register → OTP verify → reopen email). Visible to any recipient without the extension. **Removed noreply notification email**: `sendRegistrationNotification()` removed from `CryptoServiceImpl` — unregistered recipients now receive only the encrypted email (footer is sufficient). **Friendly error message**: catch block in `Encrypt & Send` handler now detects `"Key not found"` / `"not found"` errors and shows "You are not registered yet. Click the SeQureMail extension icon to register first." instead of raw server error. **Docs updated**: `seqremail-design.md` bumped to v2.1 (revision history, design decisions, component map SMTP note, encrypt flow, edge cases table, envelope format section), `SETUP.md` updated to v3 (Step 3 rewritten for OTP flow, Step 5 removed, send step updated, troubleshooting refreshed). Multiple fresh tests run (DB truncated via Docker exec).
- **Time Spent**: ~1 hour

### 2026-06-25 (Session 8) - Documentation Full Update
- **Changes**: Rewrote all 7 documentation files to reflect POC v3. `seqremail-design.md` v2.0 complete rewrite — all sections (components, data flows, security model, envelope format). All 5 SDD proposal sections updated. ASCII + Mermaid diagrams, threat model T1–T10, OTP properties, ECDH double-envelope properties, POC vs production gap table, structured decrypted view UX, parseEnvelope() zero-width char handling.
- **Time Spent**: ~1.5 hours

### 2026-06-25 (Session 7) - Fresh Test Protocol + Timezone Fix
- **Changes**: Defined "fresh test" protocol (TRUNCATE user_keys + otp_verifications + clear chrome.storage.local + reload extension). Gmail inbox label issue diagnosed as Gmail account setting (not extension bug). Fixed DB timezone: `TZ=Asia/Kuala_Lumpur` in both docker-compose services, `serverTimezone=Asia/Kuala_Lumpur`. Containers restarted — DB now shows MYT timestamps.
- **Time Spent**: ~30 min

### 2026-06-25 (Session 6) - Attachment Auto-Fetch Decryption
- **Changes**: Replaced manual file-picker with auto-fetch from Gmail DOM (`findGmailAttachmentUrl`). Images rendered inline; all other files via `chrome.downloads.download()` with correct filename+extension. Added `"downloads"` permission to manifest. Fallback to file picker if link not in DOM.
- **Time Spent**: ~1 hour

### 2026-06-25 (Session 5) - Double-Envelope + Claimed Flag
- **Changes**: Double-envelope: `recipient` + `sender` blocks, independent ephemeral keypairs. Block selection via `getCurrentGmailEmail()` (document.title parse). Claimed flag: V4 Flyway migration, auto-provisioned keys blocked until OTP (`claimed=false→true`). `generateKeyPair()` idempotent. All scenarios verified: receiver blocked before claim ✅, decrypts after OTP ✅, sender decrypts via sender block ✅, stranger blocked ✅.
- **Time Spent**: ~2.5 hours

## Historical Summary
Project started 2026-05-21 as a Chrome MV3 POC to prove client-side email encryption via Gmail DOM interception. Over 14+ sessions spanning one month, evolved from shared passphrase (Level 0) → RSA-OAEP keypair (Level 1) → ECDH P-256 client-side → ECDH P-256 server-side HSM with full security model + attachment support. Key milestones: (1) Full SDD written 2026-05-26; (2) SDD formalised 2026-06-11; (3) POC redesigned to Level 1 2026-06-15 — DOM-based, no OAuth2; (4) Key API backend live 2026-06-15; (5) ECDH P-256 migration 2026-06-18; (6) OTP email verification 2026-06-24; (7) Server-side HSM 2026-06-24; (8) Auto-provision receiver keypair 2026-06-24; (9) Double-envelope + claimed flag 2026-06-25 — full security model; (10) Attachment auto-fetch 2026-06-25; (11) Documentation full update 2026-06-25 — all docs rewritten to POC v3; (12) Registration footer + noreply email removed + friendly error UX 2026-06-28.

## Technical Notes
- **Repository**: Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\extension\` | API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\key-api\`
- **Design Doc (POC)**: `C:\PROJECTS\SEQURE MAIL\Documentation\POC\seqremail-poc-design-v2.md` (v1.5)
- **Key Dependencies**: Chrome Extension MV3, SeQureMail Key API (Spring Boot + MySQL), JavaMailSender (SMTP), Web Crypto API (file-only)
- **Crypto Plan**:
  - ~~v1: RSA-OAEP-2048~~ | ~~v2: ECDH P-256 client-side~~ | **v3 (current): SERVER-ECDH-P256-AES-256-GCM double-envelope** | v4 planned: ML-KEM-768
- **Envelope Format v3 (double)**: `seqremail: poc-v3`, `from`, `to`, `recipient: {ephemeralPublicKey, iv, ciphertext}`, `sender: {ephemeralPublicKey, iv, ciphertext}`
- **Security Model**: Auto-provision (`claimed=false`) → receiver registers via OTP → `claimed=true` → decrypt unlocked. Each party decrypts with own private key via own block. Strangers blocked by `selectBlock()` check.
- **Gmail Account Detection**: `getCurrentGmailEmail()` parses `document.title` ("Inbox - email@gmail.com - Gmail") to determine active account for block selection.
- **Flyway Migrations**: V1 (user_keys) | V2 (otp_verifications) | V3 (private_key) | V4 (claimed flag) | V5 (role)
- **Team Feedback Checklist** (branching strategy: one branch per feature, merge to master sequentially):
  - [x] #7 Non-subscriber role — `feature/7-non-subscriber-role` ✅ pushed
  - [ ] #6 User management — charge per user (depends on #7)
  - [ ] #5 HSM integration
  - [ ] #4 Outlook Web (OWA) support
  - [ ] #2 Firefox support
  - [ ] #3 Safari support
  - [ ] #1 MyDigital ID onboarding
  - [ ] #8 Outlook plugin (if possible)

---
**Last Updated**: 2026-07-01 (session 10) | **Position**: #1/10 Active
