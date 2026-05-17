# MyTrustID Desktop - WPF Desktop App for Token/SoftCert Login & Signing
*WPF .NET Framework 4.8 desktop application with WebSocket server integration*

## Project Overview
- **Type**: Desktop Application (WPF)
- **Period**: 2026-03-31 - Active
- **Tech Stack**: C# + WPF + .NET Framework 4.8 + WebSocket
- **Completion**: 97%
- **Due Date**: 2026-04-15
- **Duration**: ~8.75 hours

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
- **Last Session**: 2026-05-17 - Chrome LNA deep-dive: nested iframe diagnosis, browser instruction box, bpfk findings doc
- **Next Steps**: bpfk team adds `allow="local-network-access"` to their outer iframe; rebuild MTID Desktop with HttpService.cs headers; deploy updated JSPs to production; include auth.html in installer
- **Completed**: STA thread fix ✓, NullRef fix ✓, expired cert fallback ✓, CertVerifier integrated ✓, NPRA 5-param auth ✓, BPFKCert fix ✓, error codes verified ✓, UI freeze async fix ✓, Chrome PNA iframe fix ✓, JSON parse bug fixed ✓, LNA instruction box on JSPs ✓
- **Known Issues**: bpfk outer iframe missing `allow="local-network-access"` — confirmed via Sec-Fetch-Dest + empty MTID header log; MTID Desktop needs rebuild for HttpService.cs LNA headers

## Key Logic Notes
- `CheckBNMcert()` in `PickupNewCertViewModel.cs:507` — reads X509 SubjectDN, checks `O=BNM`
- Returns `isModifiedCka = true` if BNM cert detected → affects how cert is imported via `ImportCertificate()`
- Chrome LNA silent block signature: TLS opens, 22-sec gap, empty `Header start:` in MTID log, connection closes

## Session History (Last 5)

### 2026-05-15 to 2026-05-17 - Chrome LNA Nested Iframe Diagnosis & JSP Hardening
- **Changes**: Diagnosed JSON parse bug in page_auth.jsp — regex `/[ -]+/g` was stripping `"` chars from JSON, breaking JSON.parse silently. Fixed by removing regex, using direct `JSON.parse(msg.data)` + `encodeURIComponent` on all redirect params. Added LNA headers to `HttpService.cs` RespondSuccess() — `Access-Control-Allow-Private-Network: true`, `Permissions-Policy: local-network-access=*`, `Access-Control-Allow-Origin: *`. Added `allow="local-network-access"` to page_auth.jsp iframe. Confirmed production failure: bpfk uses iframe (Sec-Fetch-Dest: iframe), their outer iframe missing `allow` → Chrome does silent TLS probe (22-sec gap, empty header in MTID log). Modified testq3.php to simulate bpfk iframe with `allow="local-network-access"` + `target="pki_frame"`. Fixed error_page_auth.jsp `target="_top"` for callback. Created LNA findings doc (`LNA_Findings_Draft.txt`). Added browser-detecting LNA instruction box (Chrome/Edge > 138 only, soft grey) to page_auth.jsp, page_auth_jnlp.jsp, page_auth_jnlp_ORI.jsp. Drafted bpfk email template.
- **Time Spent**: ~3 hours

### 2026-05-14 - Chrome PNA Fix — iframe+postMessage Architecture
- **Changes**: Diagnosed Chrome PNA blocking `wss://mtid.msctrustgate.com/NpraAuth`. Created iframe+postMessage architecture: `auth.html` served by MTID Desktop, loads WebSocket from same origin. Updated auth.html to read 5 params from URL query string. Updated bpfk.jsp with hidden iframe + params in src URL. Fixed `HttpService.cs` crash (illegal chars in path — query string stripping). Fixed OPTIONS regex in `WebServer.cs`.
- **Time Spent**: ~2 hours

### 2026-05-12 - FaultException Investigation
- **Changes**: Investigated `FaultException: Version can not be null` from `AutoUpdate.VerseCheck()`. Config correct — user confirmed own error, no code changes.
- **Time Spent**: ~15 min

### 2026-05-08 - AboutViewModel Pattern Review
- **Changes**: Reviewed `AboutViewModel` constructor — confirmed `Task.Run()` fire-and-forget pattern correct.
- **Time Spent**: ~15 min

### 2026-05-08 - UI Freeze Fix Session 2 (LoginViewModel + SoftCertViewModel)
- **Changes**: Fixed "Not Responding" on `LoginViewModel` and `SoftCertViewModel` — async/await, `_isProcessing` guard, `RelayCommand canExecute`, `Mouse.OverrideCursor`.
- **Time Spent**: ~30 min

## Historical Summary
Earlier sessions (2026-03-31 to 2026-04-21): Project registered. Full solution explored. Admin testing completed, BNM cert pickup success. May 6-7: STA thread fix, NullRef fix, expired cert fallback loop, CertVerifierWSClient integrated, NPRA 5-param auth, Java AES/CBC/NoPadding decrypt, `userSERIALNUMBER` overwrite bug fixed, BPFKCert single-pass parse fix, 10 error codes audited. May 8: UI freeze async fix (SelectStorageViewModel + PickupNewCertViewModel).

## Technical Notes
- **Repository**: TBD
- **Key Dependencies**: .NET Framework 4.8, WPF, WebSocket library
- **LNA Fix files**: `page_auth.jsp`, `page_auth_jnlp.jsp`, `page_auth_jnlp_ORI.jsp`, `error_page_auth.jsp`, `HttpService.cs`, `testq3.php`

---
**Last Updated**: 2026-05-17 (Session 9) | **Position**: #1/10 Active
