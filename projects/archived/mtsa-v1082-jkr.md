# MTSA v1.0.8.2-JKR - MyTrustSigner Agent (MyGPKI-SKALA Platform)
*Java EE web application for PDF digital signing via MyGPKI/SKALA roaming certificate infrastructure*

## Project Overview
- **Type**: Web Application (Java EE / Servlet)
- **Period**: 2026-03-17 - 2026-04-22
- **Tech Stack**: Java 17 + iText5 + BouncyCastle + NetBeans + Tomcat
- **Completion**: 100%
- **Duration**: ~6.5 hours
- **Status**: Completed — Delivered to client 2026-04-22

## Current Status
- **Last Session**: 2026-04-22 - deploy.sh improved (directory checks, copy_if_missing helper, validator properties)
- **Next Steps**: None — project delivered
- **Known Issues**: None

## Session History (Last 5)

### 2026-04-22 - deploy.sh Improvements & Delivery
- **Changes**:
  - deploy.sh: expanded directory creation from 5 to 12 dirs (audit, files, signedfiles, data, jobs, docs, tmp added)
  - deploy.sh: added `copy_if_missing` helper — copies config/data files on first install, silently skips if already present
  - deploy.sh: added validator.properties + validator.inputlength.properties to install flow
  - deploy.sh: added wscredentials.xml to install flow
  - Project delivered to client
- **Time Spent**: ~30 min

### 2026-04-22 - Project Resumed
- **Changes**: Project loaded into active session. Pending delivery to client — awaiting PM notification.
- **Time Spent**: ~0 min

### 2026-04-10 - Async Signing & Resource Leak Fix
- **Changes**:
  - Fixed DsJobWatcher.java: WatchService + FileInputStream both now wrapped in try-with-resources
  - Added `volatile` to SigningSession.java fields for cross-thread visibility
  - NEW: SigningJobExecutor.java — singleton bounded ThreadPoolExecutor (queue 500, AbortPolicy → 503 on full)
  - NEW: SignStatus.java — GET `/sign/status?sessionId=xxx` servlet
  - ExternalSign.java: replaced blocking 120s signing call with async submit → 202 Accepted → browser polls
  - signing_external.jsp: two-phase JS polling flow
  - NEW: docs/signing-flow.txt + docs/server-spec.txt
- **Time Spent**: ~120 min

### 2026-04-02 - Signing Flow Bug Fixes & Features
- **Changes**: Fixed "Cuba Semula" button, OTP code-25 handling, debug panel toggle feature, switch-env.ps1 improvements.
- **Time Spent**: ~90 min

### 2026-03-30 - INTERNAL Signing Fix Complete
- **Changes**: Double SHA256 hashing bug fixed. OTP code 25 fix. API doc updated to v1.4.1.
- **Time Spent**: ~90 min

## Historical Summary
Project started 2026-03-17 as MTSA signing platform maintenance for JKR client. Focus on MyGPKI roaming signing flow (INTERNAL + ROAMING paths), PDF deferred signing with iText5, and CMS SignedData via BouncyCastle. Key milestones: signKeyword null bug fixed, double SHA256 hashing bug fixed, OTP code-25 retry flow fixed, async signing with ThreadPoolExecutor, Docker deploy.sh improvements. Delivered to client 2026-04-22.

## Technical Notes
- **Repository**: Git (committed & pushed as of 2026-03-30)
- **Key Dependencies**: iText5, BouncyCastle, Java EE Servlet API
- **Dev Path**: `C:\PROJECTS\MyGPKI-SKALA\Development\mtsa\`
- **Signing Flow**: PDF_prepareHash → GetSignHash → (external signing) → PDF_embedSignature → SignComplete
- **Config File**: `PILOT/mtsa_jkr/config/gpki.properties`
- **Env Switch**: `switch-env.ps1 localhost|sandbox|production`

---
**Last Updated**: 2026-04-22 | **Status**: Completed — Archived
