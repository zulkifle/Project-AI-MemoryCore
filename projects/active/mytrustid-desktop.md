# MyTrustID Desktop - WPF Desktop App for Token/SoftCert Login & Signing
*WPF .NET Framework 4.8 desktop application with WebSocket server integration*

## Project Overview
- **Type**: Desktop Application (WPF)
- **Period**: 2026-03-31 - Active
- **Tech Stack**: C# + WPF + .NET Framework 4.8 + WebSocket
- **Completion**: 97%
- **Due Date**: 2026-04-15
- **Duration**: ~11.25 hours

## Solution Structure
| Project | Purpose | Path |
|---------|---------|------|
| **MyTrustIDv1** | Main application code | `C:\repos\MyTrustIDv1_AATL-GENERIC\MyTrustIDv1\` |
| **Tester** | Testing purpose | `C:\repos\MyTrustIDv1_AATL-GENERIC\Tester\` |
| **WebSockets** | WebSocket server — external repo referenced in solution | `C:\repos\WebSocketslog4net\WebSockets\` |
| **MyTrustID** | Visual Studio Installer project | `C:\repos\MyTrustIDv1_AATL-GENERIC\MyTrustID\` |

## Key Features
- Login / Signing using **Token** or **SoftCert**
- **WebSocket server API** — browser calls in via WebSocket client
- Desktop installer via VS installer project

## Current Status
- **Last Session**: 2026-05-21 - UI lock during processing (all screens) + installer bat overhaul + code signing tool
- **Next Steps**: bpfk team adds `allow="local-network-access"` to their outer iframe; rebuild MTID Desktop with HttpService.cs headers; deploy updated JSPs to production; sign MyTrustID.EXE + MSI with real SafeNet token when cert arrives
- **Completed**: STA thread fix ✓, NullRef fix ✓, expired cert fallback ✓, CertVerifier integrated ✓, NPRA 5-param auth ✓, BPFKCert fix ✓, error codes verified ✓, UI freeze async fix ✓, JSON parse bug fixed ✓, LNA instruction box on JSPs ✓, LNA findings doc drafted ✓, trim fix (UserID+UUID) ✓, UI lock all screens ✓, installer bat overhaul ✓
- **Known Issues**: bpfk outer iframe missing `allow="local-network-access"` — confirmed via Sec-Fetch-Dest + empty MTID header log; MTID Desktop needs rebuild for HttpService.cs LNA headers

## Key Logic Notes
- `CheckBNMcert()` in `PickupNewCertViewModel.cs:507` — reads X509 SubjectDN, checks `O=BNM`
- Returns `isModifiedCka = true` if BNM cert detected → affects how cert is imported via `ImportCertificate()`
- Chrome LNA silent block signature: TLS opens, 22-sec gap, empty `Header start:` in MTID log, connection closes

## Session History (Last 5)

### 2026-05-21 - UI Lock All Screens + Installer Bat Overhaul + Code Signing Tool
- **Changes**: (1) Added `IsNotProcessing` + `NavEnabled` pattern across all 4 VMs — `PickupNewCertViewModel`, `SelectStorageViewModel`, `LoginViewModel`, `SoftCertViewModel` — locks all input fields, radio buttons, and sidebar nav during processing, re-enables in `finally`. (2) Rewrote `mytrustid.bat`: replaced dynamic .vbs→PowerShell shortcut, bitsadmin→WebClient.DownloadFile, .NET 4.6.1→4.8 (Release DWORD check), WMIC→registry + direct `msiexec /x` (synchronous), fixed auto-run race condition. (3) Created `sign.bat` — double-click code signing tool for SafeNet token using `signtool /a`.
- **Time Spent**: ~2 hours

### 2026-05-20 - PickupNewCertViewModel Trim Fix
- **Changes**: Added `UserID = UserID?.Trim()` and `UUID = UUID?.Trim()` in `PickupCert()` at line 184, before `ValidateParam` call. Fixes whitespace issue when users paste activation code from email. Both fields trimmed before validation and all downstream logic.
- **Time Spent**: ~10 min

### 2026-05-17 - LNA Findings Doc Revision & auth.html Revert
- **Changes**: Reverted auth.html iframe+postMessage approach — deleted `auth.html` from `bin\Debug`. Revised `LNA_Findings_Draft.txt`: added JNLP (Gen 1) vs MTID Desktop (Gen 2) background framing; corrected root cause to blame Chrome 142 LNA policy enforcement (not missing attribute — the attribute is the resolution); updated affected components to list both generations; repositioned MTID Desktop as the mitigation plan for JNLP deprecation. Added NOTE ON JNLP clarifying migration to MTID alone does not bypass LNA.
- **Time Spent**: ~30 min

### 2026-05-15 to 2026-05-17 - Chrome LNA Nested Iframe Diagnosis & JSP Hardening
- **Changes**: Diagnosed JSON parse bug in page_auth.jsp — regex `/[ -]+/g` stripping `"` from JSON, breaking JSON.parse silently. Fixed by removing regex, using direct `JSON.parse(msg.data)`. Added LNA headers to `HttpService.cs` RespondSuccess(). Confirmed production failure: bpfk uses iframe (Sec-Fetch-Dest: iframe), outer iframe missing `allow` → Chrome silent TLS probe (22-sec gap, empty header in MTID log). Modified testq3.php to simulate bpfk iframe. Fixed error_page_auth.jsp `target="_top"`. Created `LNA_Findings_Draft.txt`. Added browser-detecting LNA instruction box (Chrome/Edge > 138 only, soft grey) to page_auth.jsp, page_auth_jnlp.jsp. Drafted bpfk email template.
- **Time Spent**: ~3 hours

### 2026-05-14 - Chrome PNA Investigation — HttpService.cs Fix
- **Changes**: Diagnosed Chrome PNA blocking `wss://mtid.msctrustgate.com/NpraAuth`. Fixed `HttpService.cs` crash (illegal chars in path — query string stripping). Fixed OPTIONS regex in `WebServer.cs`. Explored iframe+postMessage approach (later reverted).
- **Time Spent**: ~2 hours

### 2026-05-12 - FaultException Investigation
- **Changes**: Investigated `FaultException: Version can not be null` from `AutoUpdate.VerseCheck()`. Config correct — user confirmed own error, no code changes.
- **Time Spent**: ~15 min

### 2026-05-08 - UI Freeze & Pattern Review
- **Changes**: Fixed "Not Responding" on `LoginViewModel` and `SoftCertViewModel` — async/await, `_isProcessing` guard, `RelayCommand canExecute`, `Mouse.OverrideCursor`. Reviewed `AboutViewModel` constructor — confirmed `Task.Run()` fire-and-forget pattern correct.
- **Time Spent**: ~45 min

## Historical Summary
Earlier sessions (2026-03-31 to 2026-04-21): Project registered. Full solution explored. Admin testing completed, BNM cert pickup success. May 6-7: STA thread fix, NullRef fix, expired cert fallback loop, CertVerifierWSClient integrated, NPRA 5-param auth, Java AES/CBC/NoPadding decrypt, `userSERIALNUMBER` overwrite bug fixed, BPFKCert single-pass parse fix, 10 error codes audited. May 8: UI freeze async fix (SelectStorageViewModel + PickupNewCertViewModel).

## Technical Notes
- **Repository**: TBD
- **Key Dependencies**: .NET Framework 4.8, WPF, WebSocket library
- **LNA Fix files**: `page_auth.jsp`, `page_auth_jnlp.jsp`, `page_auth_jnlp_ORI.jsp`, `error_page_auth.jsp`, `HttpService.cs`, `testq3.php`

---
**Last Updated**: 2026-05-21 (Session 11) | **Position**: #1/10 Active
