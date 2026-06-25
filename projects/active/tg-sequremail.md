# TG SeQureMail - End-to-End Encrypted Gmail Extension
*Chrome extension POC for server-side HSM encrypted email via Gmail DOM interception*

## Project Overview
- **Type**: Chrome Extension (Manifest V3)
- **Client**: Any Gmail user (external-facing POC)
- **Period**: 2026-05-21 - Active
- **Tech Stack**: Frontend: Chrome Extension MV3 (thin client) | Backend: SeQureMail Key API (Spring Boot 3.2 + MySQL + Flyway) — Software HSM | Crypto: ECDH P-256 + AES-256-GCM double-envelope (server-side, pure JDK) | Auth: OTP email verification (JavaMailSender)
- **Completion**: 99%
- **Duration**: 15 hours
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-06-25 (session 7) - Fresh test + Gmail inbox investigation + DB timezone fix
- **Next Steps**:
  1. Full E2E test — register both accounts, encrypt & send, decrypt on receiver side
  2. Confirm attachment decrypt still works after fresh registration
- **Known Issues**: None

## Session History (Last 5)

### 2026-06-25 (Session 7) - Fresh Test Protocol + Timezone Fix
- **Changes**: Defined "fresh test" protocol (DELETE user_keys + otp_verifications + clear chrome.storage.local + reload extension) — saved to Jessy memory. Investigated Gmail inbox label issue — diagnosed as Gmail account "Inbox type" setting, not an extension bug. Fixed DB timezone: added `TZ=Asia/Kuala_Lumpur` to both `app` and `db` services in `docker-compose.yml`, changed `serverTimezone=UTC` → `serverTimezone=Asia/Kuala_Lumpur`. Containers restarted — DB now shows MYT (UTC+8) timestamps.
- **Time Spent**: ~30 min

### 2026-06-25 (Session 6) - Attachment Auto-Fetch Decryption
- **Changes**: Replaced manual file-picker attachment decrypt with auto-fetch from Gmail DOM. `findGmailAttachmentUrl(encryptedName, msgEl)` searches `a[href*="view=att"]` anchors inside the message element, matches by filename text — same approach as PQCee. `fetch()` uses active Gmail session (content script runs on mail.google.com). Fallback to file picker if link not found (e.g. attachment panel not expanded). **Image files**: rendered inline below attachment row via `<img>` tag + `.sqm-attach-preview` div. **All other files**: routed through background.js `SQM_DOWNLOAD` message → `chrome.downloads.download()` with correct filename+extension — fixes Chrome's built-in viewer (PDF) intercepting blob URLs and losing the filename extension. Added `"downloads"` permission to `manifest.json`. `background.js` decodes base64 payload → data URL → `chrome.downloads.download({ url, filename, saveAs: false })`. All file types work: images inline, PDF/DOCX/ZIP/etc auto-download with correct filename.
- **Time Spent**: ~1 hour

### 2026-06-25 (Session 5) - Double-Envelope + Claimed Flag
- **Changes**: **Double-envelope**: `CryptoServiceImpl.encrypt()` now encrypts plaintext TWICE — `recipient` block (receiver's public key + ephemeral) and `sender` block (sender's public key + ephemeral). Each party decrypts with their own private key. `EncryptRequest` + `CryptoService.encrypt()` signature updated to include `senderEmail`. `CryptoController.encrypt()` passes both emails. `crypto.js` + `content_script.js` updated to pass `senderEmail` from `sqmEmail`. **Block selection**: `handleDecrypt` uses `getCurrentGmailEmail()` (reads `document.title` — format: "Inbox - email@gmail.com - Gmail") to detect active Gmail account; compares against `envelope.from` to pick sender or recipient block. **Claimed flag**: V4 Flyway migration (`claimed BOOLEAN DEFAULT FALSE`), `UserKey.claimed` field, `CryptoService.claimKeyPair()` + `CryptoServiceImpl` implementation. Auto-provisioned keys have `claimed=false` — decrypt blocked until OTP registration sets `claimed=true`. `CryptoController.generate()` calls `claimKeyPair()` after keypair creation. `decrypt()` rejects unclaimed keys with 400. **Other fixes**: sender-registration guard in encrypt onclick, modal text updated ("Server-side HSM"), `generateKeyPair()` made idempotent (existing keypair returns unchanged). All verified: receiver blocked before claim ✅, receiver decrypts after OTP ✅, sender decrypts via sender block ✅, stranger blocked ✅.
- **Time Spent**: ~2.5 hours

### 2026-06-24 (Session 4) - Auto-Provision Receiver Keypair
- **Changes**: `CryptoServiceImpl.encrypt()` auto-generates EC P-256 keypair for unregistered recipients, sends best-effort notification email. `generateKeyPair()` made idempotent (returns existing if complete keypair present). Fixed missing `OtpException` import. Docker rebuilt + verified.
- **Time Spent**: ~30 min

### 2026-06-24 (Session 3) - Server-Side HSM Architecture
- **Changes**: Full pivot — all crypto moved to Key API server. V3 migration (`private_key TEXT`), `CryptoService` + `CryptoServiceImpl` (pure JDK ECDH P-256 + AES-256-GCM), `CryptoController` (generate/encrypt/decrypt). Extension rewritten as thin client: `background.js`, `keystore.js`, `crypto.js`, `content_script.js`, `popup.js`. Docker rebuilt + E2E verified.
- **Time Spent**: ~2.5 hours

### 2026-06-24 (Session 2) - OTP Email Verification
- **Changes**: Full OTP flow — `OtpService` + `OtpServiceImpl` (JavaMailSender, SecureRandom, cooldown, upsert, UUID token). `KeyController` + `GlobalExceptionHandler` updated. Extension: 3-step popup flow, `sqmOtpPending` persisted to storage for popup-close recovery.
- **Time Spent**: ~2 hours

## Historical Summary
Project started 2026-05-21 as a Chrome MV3 POC to prove client-side email encryption via Gmail DOM interception. Over 13+ sessions spanning one month, evolved from shared passphrase (Level 0) → RSA-OAEP keypair (Level 1) → ECDH P-256 client-side → ECDH P-256 server-side HSM with full security model + attachment support. Key milestones: (1) Full SDD written 2026-05-26; (2) SDD formalised 2026-06-11; (3) POC redesigned to Level 1 2026-06-15 — DOM-based, no OAuth2; (4) Key API backend live 2026-06-15; (5) ECDH P-256 migration 2026-06-18 (RSA→ECDH, forward secrecy, DEK in body); (6) OTP email verification 2026-06-24; (7) Server-side HSM 2026-06-24; (8) Auto-provision receiver keypair 2026-06-24; (9) Double-envelope + claimed flag 2026-06-25 — full security model; (10) Attachment auto-fetch 2026-06-25 — Gmail DOM fetch, images inline, all file types download with correct filename via chrome.downloads API.

## Technical Notes
- **Repository**: Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\extension\` | API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\key-api\`
- **Design Doc (POC)**: `C:\PROJECTS\SEQURE MAIL\Documentation\POC\seqremail-poc-design-v2.md` (v1.5)
- **Key Dependencies**: Chrome Extension MV3, SeQureMail Key API (Spring Boot + MySQL), JavaMailSender (SMTP), Web Crypto API (file-only)
- **Crypto Plan**:
  - ~~v1: RSA-OAEP-2048~~ | ~~v2: ECDH P-256 client-side~~ | **v3 (current): SERVER-ECDH-P256-AES-256-GCM double-envelope** | v4 planned: ML-KEM-768
- **Envelope Format v3 (double)**: `seqremail: poc-v3`, `from`, `to`, `recipient: {ephemeralPublicKey, iv, ciphertext}`, `sender: {ephemeralPublicKey, iv, ciphertext}`
- **Security Model**: Auto-provision (`claimed=false`) → receiver registers via OTP → `claimed=true` → decrypt unlocked. Each party decrypts with own private key via own block. Strangers blocked by `selectBlock()` check.
- **Gmail Account Detection**: `getCurrentGmailEmail()` parses `document.title` ("Inbox - email@gmail.com - Gmail") to determine active account for block selection.
- **Flyway Migrations**: V1 (user_keys) | V2 (otp_verifications) | V3 (private_key) | V4 (claimed flag)

---
**Last Updated**: 2026-06-25 (session 7) | **Position**: #1/10 Active
