# TG SecureMail - End-to-End Encrypted Gmail Extension
*Chrome extension POC for client-side email encryption/decryption via Gmail API*

## Project Overview
- **Type**: Chrome Extension (Manifest V3)
- **Client**: Internal
- **Period**: 2026-05-21 - Active
- **Tech Stack**: Backend: Gmail API (Google) | Frontend: Chrome Extension MV3 + JavaScript | Storage: chrome.storage (local key storage)
- **Completion**: 0%
- **Duration**: 75 min
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-05-21 - Project scoped + skill system prepared
- **Next Steps**:
  1. Scaffold Chrome extension (Manifest V3 structure)
  2. Set up Gmail API OAuth2 integration (send + read scope)
  3. Implement AES-based client-side encrypt/decrypt layer (Web Crypto API)
  4. Basic public/private key exchange model (key generation + sharing flow)
  5. Gmail-embedded UI: Compose → Encrypt → Send | Receive → Decrypt actions
  6. End-to-end POC test: full flow from compose to decrypted read
- **Known Issues**: None

## Session History (Last 5)

### 2026-05-21 - Off-Project Debug Session (DigitalSeal + Python Setup)
- **Changes**: Off-project — debugged DigitalSeal 0-byte output issue (e-Court client PDF with broken font structure: Arial FirstChar=LastChar=32, orphaned FontFile objects). Root cause confirmed via PDF binary analysis. Installed Python 3.12 + pikepdf as debugging toolchain for future PDF investigations.
- **Time Spent**: ~45 min

### 2026-05-21 - Project Scoped + Skill System Prepared
- **Changes**: Project created and full scope defined (Chrome MV3 + Gmail API + AES-256-GCM + key management POC). Absorbed `brainstorming` and `verification-before-completion` skills from superpowers into jessy-skills plugin — ready to use for TG SecureMail design phase. MyTrustID .bat: removed CertData.xml + LicInfo.xml deletion lines.
- **Time Spent**: ~30 min

### 2026-05-21 - Project Created
- **Changes**: Initial project setup. Scope defined as working POC — Chrome MV3 + Gmail API + AES encryption + basic key management. Post-quantum extensibility planned but out of scope for MVP.
- **Time Spent**: ~0 min

## Historical Summary
[No history yet — this section is populated when session count exceeds 5]

## Technical Notes
- **Repository**: TBD
- **Key Dependencies**: Chrome Extension Manifest V3, Gmail API (OAuth2), Web Crypto API (AES-GCM), future: post-quantum crypto lib (e.g., liboqs or PQClean WASM port)
- **Scope**: POC only — validate end-to-end flow (compose → encrypt → send → receive → decrypt)
- **Crypto Plan**: MVP uses AES-256-GCM (symmetric, key wrapped via RSA or ECDH). Post-quantum (e.g., Kyber/ML-KEM) planned as a future extension.

---
**Last Updated**: 2026-05-21 (Session 3) | **Position**: #1/10 Active
