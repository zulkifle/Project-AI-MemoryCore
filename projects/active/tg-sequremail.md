# TG SeQureMail - End-to-End Encrypted Gmail Extension
*Chrome extension POC for client-side email encryption/decryption via Gmail API*

## Project Overview
- **Type**: Chrome Extension (Manifest V3)
- **Client**: Any Gmail user (external-facing POC)
- **Period**: 2026-05-21 - Active
- **Tech Stack**: Frontend: Chrome Extension MV3 + Web Crypto API (ECDH P-256, AES-256-GCM), DOM-based (no Gmail API) | Backend: SeQureMail Key API (Spring Boot 3.2 + MySQL + Flyway) | Auth: none for POC (open Key API) | Trust: planned TGCA for production
- **Completion**: 78%
- **Duration**: 6.5 hours
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-06-24 - OTP email verification scaffolding (partial — session cut by API 500)
- **Next Steps**:
  1. Implement `OtpService` interface + `OtpServiceImpl` (send OTP via JavaMailSender, verify OTP + token)
  2. Add `POST /api/keys/request-otp` and `POST /api/keys/verify-otp` endpoints to `KeyController`
  3. Update `GlobalExceptionHandler` — `OtpException` → 400, `TooManyRequestsException` → 429
  4. After API complete: reload extension, E2E re-test compose → encrypt → send → decrypt
- **Known Issues**: OtpService + controller endpoints not yet written — API won't compile until these are added

## Session History (Last 5)

### 2026-06-24 - OTP Email Verification Scaffolding (Partial)
- **Changes**: Started OTP email verification feature for Key API. Added `spring-boot-starter-mail` dep to pom.xml. Added SMTP (smtp.gmail.com:587) + OTP config (expiry 120s, cooldown 60s, max 3 attempts, token 600s) to application.properties. Created Flyway `V2__add_otp_verifications.sql` (otp_verifications table: id, email, otp_code, attempts, last_otp_sent_at, otp_expires_at, verified_at, verification_token). Created: `OtpVerification` entity, `OtpVerificationRepository`, `OtpProperties` (ConfigurationProperties), `OtpConfig` (EnableConfigurationProperties), `OtpException`, `TooManyRequestsException`. Session cut off by API 500/529 — `OtpService`, `OtpServiceImpl`, controller endpoints, and `GlobalExceptionHandler` updates still pending.
- **Time Spent**: ~30 min

### 2026-06-18 - ECDH P-256 Migration + Documentation Update
- **Changes**: Upgraded crypto algorithm from RSA-OAEP-2048 to ECDH P-256 + AES-256-GCM. `crypto.js` completely rewritten — ephemeral keypair per email for forward secrecy, `_deriveAESKey(ECDH)` replaces RSA wrap/unwrap, file DEK embedded in encrypted body JSON (no separate `wrapKey`). `background.js`: removed `SQM_UNWRAP_DEK` handler; `ensureKeypair()` clears stale RSA artifacts (`ownPublicKeyJwk`, `ownPrivateKeyJwk`, `sqmRegistered`, `contacts`). `keystore.js`: `loadPrivateKey()` rejects `kty !== 'EC'` to block stale RSA JWK import. `manifest.json`: version bumped to 2.0, description updated to ECDH P-256. `content_script.js`: envelope format `poc-v2`, `ECDH-P256-AES-256-GCM` algorithm, UI labels updated, attachment decrypt uses `att.dekRaw` directly (no background round-trip), stale key rejection in `showSendModal`. Fixed cascade attachment encryption failure — stale RSA public keys in contacts cache caused `importPublicKey()` to throw → email never sent. Documentation: `seqremail-poc-design-v2.md` updated v1.4→v1.5 (all sections); `SETUP.md` fully rewritten (removed OAuth2 step, added Key API step, ECDH flow).
- **Time Spent**: ~2 hours

### 2026-06-15 (2) - Key API Live + Bug Fixes
- **Changes**: API confirmed working via docker compose (MySQL port 3307, Flyway runs V1 on startup). Fixed Dockerfile to use Maven Docker image (no mvnw needed). Two extension bugs fixed: (1) double Encrypt toggle — moved `sqmDone='true'` to top of `injectEncryptToggle` before retry timer; (2) decrypt JSON error — Gmail converts `"` to smart quotes corrupting the envelope — fixed by base64-encoding entire payload with `btoa()` and decoding with `atob()` in `parseEnvelope`.
- **Time Spent**: ~1 hour

### 2026-06-15 - Key API Backend + Extension Auto-Lookup
- **Changes**: Scaffolded `seqremail-key-api` — Spring Boot 3.2 + MySQL + Flyway. Full layered architecture: Controller → Service → Repository → Entity. Two endpoints: `POST /api/keys/register` (upsert own public key) and `GET /api/keys/lookup?email=x` (fetch recipient key). CORS open for Chrome extension. Docker + docker-compose included. Extension updated: content_script now calls API before manual paste; popup gets email input for one-time registration. Manifest adds `http://localhost:8080/*` host permission. No manual key sharing needed — extension auto-fetches from API.
- **Backend path**: `C:\PROJECTS\SEQURE MAIL\Development\seqremail-key-api\` (19 files)
- **Extension changes**: manifest.json (host_permissions), content_script.js (apiLookup/apiRegister/tryAutoRegister), popup.html (email section), popup.js (register flow)
- **Time Spent**: ~1 hour

### 2026-06-15 - POC Redesigned to Level 1 + Full Code Scaffold
- **Changes**: Redesigned POC from Level 0 (shared passphrase) to Level 1 (RSA-OAEP keypair, no passphrase, Gmail API). UX redesigned: keypair auto-generates silently on first Gmail load (background.js). Compose toolbar gets 🔒 Encrypt toggle. Send intercept → modal (Encrypt & Send / Send as Plaintext / Cancel). Gmail API used for both send (`messages.send`) and read (`messages.get`). Decrypt banner auto-appears on encrypted emails. Sender public key auto-saved from envelope on decrypt (enables seamless reply).
- **Design doc v2**: `C:\PROJECTS\SEQURE MAIL\Documentation\POC\seqremail-poc-design-v2.md` (v1.3)
- **Code scaffold**: `C:\PROJECTS\SEQURE MAIL\Development\seqremail-poc-v1\` — 9 files: manifest.json, background.js, crypto.js, keystore.js, gmail-api.js, content_script.js, popup.html, popup.js, icon.png + SETUP.md
- **Key design decisions**: `extractable:false` private key in IndexedDB (HSM simulation), `senderPublicKey` embedded in envelope (auto key exchange on first decrypt), `chrome.identity` bridged via background.js (not available in content scripts)
- **Next**: Set up Google Cloud OAuth2 client ID → load unpacked → test E2E
- **Time Spent**: ~3 hours

## Historical Summary
Project started 2026-05-21 as a Chrome MV3 POC to prove client-side email encryption via Gmail DOM interception. Over 8 sessions spanning one month, evolved from shared passphrase (Level 0) → RSA-OAEP keypair (Level 1) → ECDH P-256 (Level 1 v2). Key milestones: (1) Full SDD written 2026-05-26 — 5-section design, Smart Hybrid model, TGCA trust model; (2) SDD formalised 2026-06-11 with 12 Mermaid diagrams + canonical API table; (3) POC redesigned to Level 1 2026-06-15 — DOM-based send/read, no OAuth2; (4) Key API backend live 2026-06-15 (Spring Boot + MySQL + docker compose); (5) ECDH P-256 migration 2026-06-18 — RSA-OAEP removed, ephemeral keypair per email, forward secrecy. Earlier sessions: project scoped, DigitalSeal debug session (off-project, pikepdf toolchain).

## Technical Notes
- **Repository**: Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail-poc-v1\` | API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\key-api\`
- **Design Doc (POC)**: `C:\PROJECTS\SEQURE MAIL\Documentation\POC\seqremail-poc-design-v2.md` (v1.5 — current, ECDH P-256)
- **Design Doc (Full SDD)**: `C:\PROJECTS\SEQURE MAIL\Documentation\Others\2026-05-26-seqremail-design.md`
- **Key Dependencies**: Chrome Extension MV3, Web Crypto API (ECDH P-256, AES-256-GCM), SeQureMail Key API (Spring Boot + MySQL), TGCA (Trustgate CA — production), HSM (PKCS#11 — production)
- **Scope**: POC only — validate end-to-end encrypted flow (compose → encrypt → send → receive → decrypt), with forward secrecy
- **Crypto Plan**:
  - ~~v1 MVP: RSA-OAEP-2048 wraps AES-256-GCM DEK~~ (completed, superseded)
  - **v2 (current): ECDH P-256 + AES-256-GCM** — ephemeral keypair per email, forward secrecy, file DEK in encrypted body
  - v3: ML-KEM-768 / Kyber (post-quantum)
- **Envelope Format**: `poc-v2`, `ECDH-P256-AES-256-GCM`, fields: `senderPublicKey`, `ephemeralPublicKey`, `iv`, `ciphertext`. File DEKs (`dekRaw`) inside encrypted body JSON (not separately wrapped).
- **Key Migration**: `loadPrivateKey()` rejects `kty !== 'EC'`; `ensureKeypair()` auto-clears stale RSA artifacts on extension reload — seamless v1.0 → v2.0 upgrade

---
**Last Updated**: 2026-06-24 | **Position**: #1/10 Active
