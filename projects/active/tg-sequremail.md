# TG SeQureMail - End-to-End Encrypted Gmail Extension
*Chrome extension POC for server-side HSM encrypted email via Gmail DOM interception*

## Project Overview
- **Type**: Chrome Extension (Manifest V3)
- **Client**: Any Gmail user (external-facing POC)
- **Period**: 2026-05-21 - Active
- **Tech Stack**: Frontend: Chrome Extension MV3 (thin client, no local crypto) | Backend: SeQureMail Key API (Spring Boot 3.2 + MySQL + Flyway) — acts as Software HSM | Crypto: ECDH P-256 + AES-256-GCM (server-side, pure JDK) | Auth: OTP email verification (JavaMailSender)
- **Completion**: 92%
- **Duration**: 11 hours
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-06-24 (session 3) - Server-side HSM architecture complete + E2E verified
- **Next Steps**:
  1. Reload extension in `chrome://extensions`
  2. Test OTP registration → server generates keypair (verify popup shows "Registered")
  3. E2E: Compose → Encrypt (server) → Send → Decrypt (server) → view plaintext
- **Known Issues**: None — server-side HSM complete, E2E API calls all verified

## Session History (Last 5)

### 2026-06-24 (Session 3) - Server-Side HSM Architecture
- **Changes**: Full architectural pivot — all ECDH P-256 + AES-256-GCM crypto moved to Key API server (private key never touches browser). **Backend**: V3 Flyway migration (`private_key TEXT` column on `user_keys`), `UserKey.java` updated (`privateKey` field), `CryptoService` interface + `CryptoServiceImpl` (pure JDK — `KeyPairGenerator` EC secp256r1, `KeyAgreement` ECDH, SHA-256 key derivation, AES-256-GCM), `CryptoController` (`POST /api/crypto/generate`, `/encrypt`, `/decrypt`), `EncryptRequest` + `DecryptRequest` DTOs. **Extension**: `background.js` simplified (thin client, no local crypto), `keystore.js` stripped to contacts-only (no keypair storage), `crypto.js` rewritten as API wrapper (`encryptMessage`→server, `decryptMessage`→server, `encryptFile`/`decryptFile` remain local for file attachments), `content_script.js` (4 edits: removed tryAutoRegister, encrypt uses email not JWK, decrypt calls API directly, removed sender-key auto-save), `popup.js` (calls `/api/crypto/generate` after OTP verify — no browser key; Share button fetches public key from `/api/keys/lookup`; status reflects registered vs unregistered). Docker rebuilt — V3 migration applied. E2E API verified: generate ✅ encrypt ✅ decrypt ✅ (round-trip "Hello SeQureMail server-side HSM!" confirmed).
- **Time Spent**: ~2.5 hours

### 2026-06-24 (Session 2) - OTP Email Verification (Full Implementation)
- **Changes**: Completed full OTP email verification feature for Key API. **Backend**: `OtpService` + `OtpServiceImpl` (JavaMailSender, SecureRandom 6-digit code, cooldown guard, upsert, token generation on verify), `OtpRequest`/`OtpVerifyRequest` DTOs, `KeyController` updated with `POST /api/keys/request-otp` + `POST /api/keys/verify-otp`, `GlobalExceptionHandler` updated (`OtpException` → 400, `TooManyRequestsException` → 429). Docker rebuilt (port conflict with old `seqremail-key-api` containers resolved). All 4 endpoints tested: register ✅, lookup ✅, request-otp ✅, cooldown 429 ✅, wrong OTP 400 ✅. **Extension**: `popup.js` + `popup.html` — 3-step OTP registration flow. Fixed popup-close state loss by persisting `sqmOtpPending` to `chrome.storage.local`.
- **Time Spent**: ~2 hours

### 2026-06-18 - ECDH P-256 Migration + Documentation Update
- **Changes**: Upgraded crypto algorithm from RSA-OAEP-2048 to ECDH P-256 + AES-256-GCM. `crypto.js` completely rewritten — ephemeral keypair per email for forward secrecy, `_deriveAESKey(ECDH)` replaces RSA wrap/unwrap, file DEK embedded in encrypted body JSON. `background.js`: removed `SQM_UNWRAP_DEK` handler; `ensureKeypair()` clears stale RSA artifacts. `keystore.js`: `loadPrivateKey()` rejects `kty !== 'EC'`. `manifest.json`: version bumped to 2.0. `content_script.js`: envelope format `poc-v2`. Documentation: `seqremail-poc-design-v2.md` updated v1.4→v1.5; `SETUP.md` fully rewritten.
- **Time Spent**: ~2 hours

### 2026-06-15 (2) - Key API Live + Bug Fixes
- **Changes**: API confirmed working via docker compose (MySQL port 3307, Flyway V1 on startup). Fixed Dockerfile to use Maven Docker image. Two extension bugs fixed: (1) double Encrypt toggle — moved `sqmDone='true'` to top of `injectEncryptToggle`; (2) decrypt JSON error — Gmail converts `"` to smart quotes → fixed by base64-encoding entire payload with `btoa()`.
- **Time Spent**: ~1 hour

### 2026-06-15 - Key API Backend + Extension Auto-Lookup
- **Changes**: Scaffolded `seqremail-key-api` — Spring Boot 3.2 + MySQL + Flyway. Full layered architecture: Controller → Service → Repository → Entity. Two endpoints: `POST /api/keys/register` and `GET /api/keys/lookup?email=x`. CORS open for Chrome extension. Docker + docker-compose included. Extension updated for auto-fetch from API.
- **Time Spent**: ~1 hour

## Historical Summary
Project started 2026-05-21 as a Chrome MV3 POC to prove client-side email encryption via Gmail DOM interception. Over 10 sessions spanning one month, evolved from shared passphrase (Level 0) → RSA-OAEP keypair (Level 1) → ECDH P-256 client-side → ECDH P-256 server-side HSM. Key milestones: (1) Full SDD written 2026-05-26; (2) SDD formalised 2026-06-11 with 12 Mermaid diagrams; (3) POC redesigned to Level 1 2026-06-15 — DOM-based send/read, no OAuth2; (4) Key API backend live 2026-06-15; (5) ECDH P-256 migration 2026-06-18 — RSA-OAEP removed, forward secrecy; (6) OTP email verification 2026-06-24 — JavaMailSender, cooldown, popup state persistence; (7) Server-side HSM 2026-06-24 — all crypto moved to server, pure JDK implementation, extension is thin client.

## Technical Notes
- **Repository**: Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\extension\` | API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\key-api\`
- **Design Doc (POC)**: `C:\PROJECTS\SEQURE MAIL\Documentation\POC\seqremail-poc-design-v2.md` (v1.5)
- **Design Doc (Full SDD)**: `C:\PROJECTS\SEQURE MAIL\Documentation\Others\2026-05-26-seqremail-design.md`
- **Key Dependencies**: Chrome Extension MV3, SeQureMail Key API (Spring Boot + MySQL), JavaMailSender (SMTP), Web Crypto API (file-only), TGCA (Trustgate CA — production)
- **Crypto Plan**:
  - ~~v1 MVP: RSA-OAEP-2048 wraps AES-256-GCM DEK~~ (completed, superseded)
  - ~~v2 client-side: ECDH P-256 + AES-256-GCM — ephemeral keypair, forward secrecy~~ (superseded)
  - **v3 (current): SERVER-ECDH-P256-AES-256-GCM** — server-side HSM model, private key stays on Key API, extension is thin client
  - v4 planned: ML-KEM-768 / Kyber (post-quantum)
- **Envelope Format v3**: `seqremail: poc-v3`, algorithm: `SERVER-ECDH-P256-AES-256-GCM`, fields: `ephemeralPublicKey` (JWK), `iv` (base64), `ciphertext` (base64). File attachments: local AES-256-GCM, DEK embedded inside server-encrypted body.
- **HSM Design**: Key API generates EC P-256 keypair server-side after OTP verification. Public JWK returned to caller. Private JWK stored in `user_keys.private_key` (TEXT). All encrypt/decrypt operations happen on server. Extension never sees private key.
- **Flyway Migrations**: V1 (user_keys), V2 (otp_verifications), V3 (private_key column on user_keys)

---
**Last Updated**: 2026-06-24 (session 3) | **Position**: #1/10 Active
