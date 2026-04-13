# MTSA v1.0.8.2-JKR - MyTrustSigner Agent (MyGPKI-SKALA Platform)
*Java EE web application for PDF digital signing via MyGPKI/SKALA roaming certificate infrastructure*

## Project Overview
- **Type**: Web Application (Java EE / Servlet)
- **Period**: 2026-03-17 - Active
- **Tech Stack**: Java 17 + iText5 + BouncyCastle + NetBeans + Tomcat
- **Completion**: 90%
- **Due Date**: TBD
- **Duration**: ~6.5 hours

## Current Status
- **Last Session**: 2026-04-11 - Packaging & config fixes
- **Next Steps**: Test deployment → submit package to client for installation
- **Known Issues**: None — resource leaks and concurrency issues resolved

## Session History (Last 5)

### 2026-04-10 - Async Signing & Resource Leak Fix
- **Changes**:
  - Fixed DsJobWatcher.java: WatchService + FileInputStream both now wrapped in try-with-resources (root cause of "too many open files" under load)
  - Deleted dead `monitorFiles()` test methods from UkmHelper.java and JkrHelper.java
  - Added `volatile` to SigningSession.java fields (status, signedFilePath, errorMessage) for cross-thread visibility
  - AppConfig.java: added `signing_workerPoolSize` field + getter, reads `gpki.signing.workerPoolSize` (default 30)
  - NEW: SigningJobExecutor.java — singleton bounded ThreadPoolExecutor (queue 500, AbortPolicy → 503 on full)
  - NEW: SignStatus.java — GET `/sign/status?sessionId=xxx` servlet (no API key required)
  - ExternalSign.java: replaced blocking 120s signing call with async submit → 202 Accepted → browser polls
  - AppStartupListener.java: init + shutdown SigningJobExecutor on app start/stop
  - web.xml: registered SignStatus servlet at `/sign/status`
  - signing_external.jsp: two-phase JS (submitSigningJob POST → startPolling every 3s, max 60 polls)
  - NEW: docs/signing-flow.txt — full INTERNAL + EXTERNAL signing flow documentation
  - NEW: docs/server-spec.txt — server spec, JVM/Tomcat/OS tuning, Docker/K8s, scaling guide
- **Time Spent**: ~120 min

### 2026-04-02 - Signing Flow Bug Fixes & Features
- **Changes**:
  - Fixed "Cuba Semula" button — replaced `location.reload()` with `retrySign()` to reuse existing nonce
  - Added `parseOtpRemainingSeconds()` — parses expiry datetime from code-25 message, shows real remaining time
  - Added `showOtpBanner()` function (was called but undefined)
  - Added `otpExpiryTime` tracking — nonce saved to sessionStorage on OTP success
  - Added `gpki.ui.debugPanel` flag in properties to toggle debug console panel on/off
  - AppConfig.java: added `ui_debugPanel` field + `isUiDebugPanel()` getter
  - Landing.java: passes `debugPanel` attribute to JSP
  - signing.jsp: debug panel HTML + JS wrapped in JSP conditionals; no-op stub when disabled
  - switch-env.ps1: added `gpki.ui.debugPanel` per env, `index.html` label updates, `SetIndexText()` helper, fixed prod properties path to `config\`
  - Deleted stale `jkr-mtsa.prod.properties` duplicate from PRODUCTION root
- **Time Spent**: ~90 min

### 2026-03-30 - INTERNAL Signing Fix Complete
- **Changes**: Double SHA256 hashing bug fixed in PdfSigningService.java. Test comments cleaned. OTP code 25 fix applied to dev + deployment signing.jsp. API technical doc updated to v1.4.1. Built, deployed, committed & pushed.
- **Time Spent**: ~90 min

### 2026-03-17 - PDF Signing Bug Fix
- **Changes**: Investigated signKeyword null bug in PDF embed flow. Fixed field name consistency across PDF_prepareHash, PDF_embedSignature, GetSignHash. Documented full signing flow end-to-end.
- **Time Spent**: ~90 min

## Historical Summary
Project started 2026-03-17 as MTSA signing platform maintenance. Focus on MyGPKI roaming signing flow (INTERNAL + ROAMING paths), PDF deferred signing with iText5, and CMS SignedData via BouncyCastle. Key milestones: signKeyword null bug fixed, double SHA256 hashing bug fixed, OTP code-25 retry flow fixed, debug panel toggle feature added.

## Technical Notes
- **Repository**: Git (committed & pushed as of 2026-03-30)
- **Key Dependencies**: iText5, BouncyCastle, Java EE Servlet API
- **Deployment Path**: Separate from dev source — always change dev (`mtsa`) first, then deployment
- **Signing Flow**: PDF_prepareHash → GetSignHash → (external signing) → PDF_embedSignature → SignComplete
- **Config File**: `PILOT/mtsa_jkr/config/gpki.properties` — controls baseUrl, debugPanel, signing settings
- **Env Switch**: `switch-env.ps1 localhost|sandbox|production` — switches all config files at once

---
**Last Updated**: 2026-04-10 (Session 2) | **Position**: #1/10 Active
