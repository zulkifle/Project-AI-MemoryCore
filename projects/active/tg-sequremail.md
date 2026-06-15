# TG SeQureMail - End-to-End Encrypted Gmail Extension
*Chrome extension POC for client-side email encryption/decryption via Gmail API*

## Project Overview
- **Type**: Chrome Extension (Manifest V3)
- **Client**: Any Gmail user (external-facing POC)
- **Period**: 2026-05-21 - Active
- **Tech Stack**: Frontend: Chrome Extension MV3 + Web Crypto API (AES-256-GCM, RSA-OAEP), DOM-based (no Gmail API) | Backend: SeQureMail Key API (Spring Boot 3.2 + MySQL + Flyway) | Auth: none for POC (open Key API) | Trust: planned TGCA for production
- **Completion**: 65%
- **Duration**: 4 hours
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-06-15 - Key API live via docker compose, extension bugs fixed (double toggle + JSON parse error)
- **Next Steps**:
  1. Reload extension in Chrome → open Gmail → popup → register `kiflezul94@gmail.com`
  2. End-to-end test: Compose → Encrypt → Send → Decrypt
  3. Update SETUP.md (currently references old Google Cloud flow)
- **Known Issues**: None — both bugs fixed this session

## Session History (Last 5)

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

### 2026-06-11 - POC Design (Level 0 "Shared Passphrase", zero backend)
- **Changes**: Scoped a bare-minimum POC to prove only the core: encrypt body in browser → ride normal Gmail → decrypt in browser, server never sees plaintext. Stripped all backend (HSM, TGCA, OAuth2, JWT, Key API, Key Directory).
- **POC Level 0 design**: shared passphrase (agreed offline) → `PBKDF2 → AES-256-GCM`; simplified `poc-v0` envelope (salt, iv, ciphertext — no keyId/DEK); manual Send (no Gmail API); 4 files only (`manifest.json`, `content_script.js`, `crypto.js`, `icon.png`); minimal perms (`storage` + `mail.google.com`)
- **New file**: `C:\PROJECTS\SEQURE MAIL\Documentation\POC\seqremail-poc-design.md` — full POC design w/ Mermaid architecture + flows, demo/test plan, acceptance criteria, limitations→Level 1→full path
- **Level 1 (optional next)**: in-browser RSA keypairs + static public-key file/Gist — demos hybrid RSA-OAEP+AES envelope, still serverless
- **Next**: scaffold the 4 POC files (proposed location `C:\PROJECTS\SEQURE MAIL\seqremail-poc\`)
- **Time Spent**: ~30 min

### 2026-06-11 - SDD Formalised: Mermaid Diagrams + Section 0 + Consistency Fixes
- **Changes**: Reviewed full design (5 proposal sections + companion design.md). Generated 12 Mermaid diagrams and appended them into the matching proposal section files at `C:\PROJECTS\SEQURE MAIL\Documentation\Others\`:
  - S1: Smart Hybrid concept · S2: **HLD** (system architecture) + **LLD** (extension internals + API surface) · S3: 4 flows (Registration, Send, Receive, Unregistered) · S4: trust chain + auth + key-rotation state · S5: compose + decrypt flows
- **New file**: `Proposal - Section 0.txt` — SDD front-matter: doc control, purpose, audience, scope (in/out), glossary, assumptions/constraints, **canonical API contract (authoritative)**, clarifications (JWT/send/cert-validation), NFRs, references, structure
- **Consistency fixes**: standardised endpoint names across all Mermaid → `/auth/token`, `/keys/register`, `/keys/lookup`, `/keys/unwrap`, `/keys/status`, `/keys/revoke`. Resolved JWT contradiction (short-lived 15-min, single-purpose, reusable, auto-refresh) and locked send mechanism (Gmail API `gmail.send`, not DOM click)
- **Note**: legacy ASCII boxes in S2-S4 keep old endpoint names (alignment-sensitive) — now governed by §0 authoritative table; rendered Mermaid is the Word-ready figure source
- **Doc type confirmed**: Technical Design Specification (SDD, IEEE 1016 style) — not a commercial proposal
- **Time Spent**: ~30 min

### 2026-05-26 - Full System Design + Design Doc Written
- **Changes**: Full brainstorming session — 5-section design completed and approved. Smart Hybrid Decryption model confirmed (HSM unwraps DEK only, body decrypted client-side). Crypto stack locked: RSA-OAEP-2048 + AES-256-GCM (v1 MVP), ECDH P-256 (v2), ML-KEM-768 Kyber (v3). Trust model: TGCA signs all public keys, extension validates cert chain before encrypting. Auth: Gmail OAuth2 → SeQureMail JWT (15 min, single-purpose). Envelope format finalised (--BEGIN SEQREMAIL-- JSON block). UX flows designed (compose inject, thread inject, popup, inbox badges). Design doc written to C:\PROJECTS\SEQURE MAIL\Documentation\Others\2026-05-26-seqremail-design.md
- **Time Spent**: ~90 min

### 2026-05-21 - Off-Project Debug Session (DigitalSeal + Python Setup)
- **Changes**: Off-project — debugged DigitalSeal 0-byte output issue (e-Court client PDF with broken font structure: Arial FirstChar=LastChar=32, orphaned FontFile objects). Root cause confirmed via PDF binary analysis. Installed Python 3.12 + pikepdf as debugging toolchain for future PDF investigations.
- **Time Spent**: ~45 min

### 2026-05-21 - Project Scoped + Skill System Prepared
- **Changes**: Project created and full scope defined (Chrome MV3 + Gmail API + AES-256-GCM + key management POC). Absorbed `brainstorming` and `verification-before-completion` skills from superpowers into jessy-skills plugin — ready to use for TG SeQureMail design phase. MyTrustID .bat: removed CertData.xml + LicInfo.xml deletion lines.
- **Time Spent**: ~30 min

### 2026-05-21 - Project Created
- **Changes**: Initial project setup. Scope defined as working POC — Chrome MV3 + Gmail API + AES encryption + basic key management. Post-quantum extensibility planned but out of scope for MVP.
- **Time Spent**: ~0 min

## Historical Summary
[No history yet — this section is populated when session count exceeds 5]

## Technical Notes
- **Repository**: Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail-poc-v1\` | API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail-key-api\`
- **Design Doc (POC)**: `C:\PROJECTS\SEQURE MAIL\Documentation\POC\seqremail-poc-design-v2.md` (v1.3 — current)
- **Design Doc (Full SDD)**: `C:\PROJECTS\SEQURE MAIL\Documentation\Others\2026-05-26-seqremail-design.md`
- **Key Dependencies**: Chrome Extension MV3, Gmail API (OAuth2), Web Crypto API, TGCA (Trustgate CA), HSM (PKCS#11), MySQL
- **Scope**: POC only — validate end-to-end flow (compose → encrypt → send → receive → decrypt)
- **Crypto Plan**:
  - v1 MVP: RSA-OAEP-2048 wraps AES-256-GCM DEK (per-email, wiped after use)
  - v2: ECDH P-256 (smaller keys)
  - v3: ML-KEM-768 / Kyber (post-quantum)
- **Trust Model**: TGCA signs every user public key cert. Extension validates full chain (Root CA → Intermediate → User cert) before encrypting. OCSP/CRL revocation checked on each lookup.
- **Email Envelope**: Plain JSON block between `--BEGIN SEQREMAIL--` / `--END SEQREMAIL--` markers, sent as Gmail body text

---
**Last Updated**: 2026-06-15 | **Position**: #2/10 Active
