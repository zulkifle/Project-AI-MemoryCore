# TG SeQureMail - End-to-End Encrypted Gmail Extension
*Chrome extension POC for server-side HSM encrypted email via Gmail DOM interception*

## Project Overview
- **Type**: Chrome Extension (Manifest V3)
- **Client**: Any Gmail user (external-facing POC)
- **Period**: 2026-05-21 - Active
- **Tech Stack**: Frontend: Chrome Extension MV3 (thin client) | Backend: SeQureMail Key API (Spring Boot 3.2 + MySQL + Flyway) — Software HSM | Crypto: ECDH P-256 + AES-256-GCM double-envelope (server-side, pure JDK) | Auth: OTP email verification (JavaMailSender)
- **Completion**: 97%
- **Duration**: 14 hours
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-06-25 (session 5) - Double-envelope + claimed flag (full security model complete)
- **Next Steps**:
  1. Truncate DB + reload extension → full E2E test (register sender → send → receiver registers via OTP → both decrypt)
  2. Verify `getCurrentGmailEmail()` correctly picks sender vs receiver block on same browser
- **Known Issues**: None — security model complete

## Session History (Last 5)

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

### 2026-06-18 - ECDH P-256 Migration
- **Changes**: RSA-OAEP-2048 → ECDH P-256 + AES-256-GCM. `crypto.js` rewritten (ephemeral keypair per email, forward secrecy, DEK in body). Stale RSA key rejection. `manifest.json` v2.0. Docs updated (design-v2.md v1.5, SETUP.md rewritten).
- **Time Spent**: ~2 hours

## Historical Summary
Project started 2026-05-21 as a Chrome MV3 POC to prove client-side email encryption via Gmail DOM interception. Over 12+ sessions spanning one month, evolved from shared passphrase (Level 0) → RSA-OAEP keypair (Level 1) → ECDH P-256 client-side → ECDH P-256 server-side HSM with full security model. Key milestones: (1) Full SDD written 2026-05-26; (2) SDD formalised 2026-06-11; (3) POC redesigned to Level 1 2026-06-15 — DOM-based, no OAuth2; (4) Key API backend live 2026-06-15; (5) ECDH P-256 migration 2026-06-18; (6) OTP email verification 2026-06-24; (7) Server-side HSM 2026-06-24; (8) Auto-provision receiver keypair 2026-06-24; (9) Double-envelope (sender + receiver blocks) + claimed flag 2026-06-25 — full security model: each party uses own key, unclaimed keys blocked until OTP registration.

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
**Last Updated**: 2026-06-25 (session 5) | **Position**: #1/10 Active
