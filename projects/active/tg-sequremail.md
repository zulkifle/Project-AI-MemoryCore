# TG SeQureMail - End-to-End Encrypted Gmail Extension
*Chrome extension POC for client-side email encryption/decryption via Gmail API*

## Project Overview
- **Type**: Chrome Extension (Manifest V3)
- **Client**: Any Gmail user (external-facing POC)
- **Period**: 2026-05-21 - Active
- **Tech Stack**: Frontend: Chrome Extension MV3 + Web Crypto API (AES-256-GCM, RSA-OAEP) | Backend: SeQureMail Key API (Trustgate) + HSM (PKCS#11) + MySQL Key Directory | Auth: Gmail OAuth2 → SeQureMail JWT (15 min) | Trust: TGCA (Trustgate CA) signs all public keys
- **Completion**: 18%
- **Duration**: 3.5 hours
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-06-11 - Reviewed design; generated Mermaid diagrams; created SDD Section 0 + fixed cross-section inconsistencies
- **Next Steps**:
  1. Phase 1 — Scaffold Chrome Extension (MV3 structure: manifest.json, background.js, content_script.js, popup, crypto.js)
  2. Phase 2 — Auth + Key Lookup (Google OAuth2 via chrome.identity, /auth/token, /keys/lookup + cert chain validation)
  3. Phase 3 — Encrypt Flow (DEK gen, AES-256-GCM, RSA-OAEP, envelope JSON, body replace)
  4. Phase 4 — Decrypt Flow (/keys/unwrap → HSM, AES-256-GCM browser decrypt)
  5. Phase 5 — Registration + Key Management (Key API backend, HSM PKCS#11, Key Directory DB)
  6. Phase 6 — End-to-end POC test
- **Known Issues**: None — design approved, ready for implementation

## Session History (Last 5)

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
- **Repository**: TBD — C:\PROJECTS\SEQURE MAIL\ (local working directory)
- **Design Doc**: `C:\PROJECTS\SEQURE MAIL\Documentation\Others\2026-05-26-seqremail-design.md`
- **Key Dependencies**: Chrome Extension MV3, Gmail API (OAuth2), Web Crypto API, TGCA (Trustgate CA), HSM (PKCS#11), MySQL
- **Scope**: POC only — validate end-to-end flow (compose → encrypt → send → receive → decrypt)
- **Crypto Plan**:
  - v1 MVP: RSA-OAEP-2048 wraps AES-256-GCM DEK (per-email, wiped after use)
  - v2: ECDH P-256 (smaller keys)
  - v3: ML-KEM-768 / Kyber (post-quantum)
- **Trust Model**: TGCA signs every user public key cert. Extension validates full chain (Root CA → Intermediate → User cert) before encrypting. OCSP/CRL revocation checked on each lookup.
- **Email Envelope**: Plain JSON block between `--BEGIN SEQREMAIL--` / `--END SEQREMAIL--` markers, sent as Gmail body text

---
**Last Updated**: 2026-06-11 | **Position**: #2/10 Active
