# MyTrustID Desktop - WPF Desktop App for Token/SoftCert Login & Signing
*WPF .NET Framework 4.8 desktop application with WebSocket server integration*

## Project Overview
- **Type**: Desktop Application (WPF)
- **Period**: 2026-03-31 - Active
- **Tech Stack**: C# + WPF + .NET Framework 4.8 + WebSocket
- **Completion**: 97%
- **Due Date**: 2026-04-15
- **Duration**: ~5.75 hours

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
- **Last Session**: 2026-05-14 - Chrome PNA fix — iframe+postMessage architecture implemented
- **Next Steps**: Deploy auth.html to production installer; bpfk.gov.my server team deploys updated JSP with iframe approach; end-to-end Chrome test
- **Completed**: STA thread fix ✓, NullRef fix ✓, expired cert fallback ✓, CertVerifier integrated ✓, NPRA 5-param auth ✓, BPFKCert fix ✓, error codes verified ✓, UI freeze async fix (all 4 ViewModels) ✓, AboutViewModel fire-and-forget ✓, Chrome PNA iframe fix ✓
- **Known Issues**: bpfk.gov.my JSP not yet deployed (server team needed); installer does not yet include auth.html

## Key Logic Notes
- `CheckBNMcert()` in `PickupNewCertViewModel.cs:507` — reads X509 SubjectDN, checks `O=BNM`
- Returns `isModifiedCka = true` if BNM cert detected → affects how cert is imported via `ImportCertificate()`

## Session History (Last 5)

### 2026-05-14 - Chrome PNA Fix — iframe+postMessage Architecture
- **Changes**: Diagnosed Chrome Private Network Access (PNA) blocking `wss://mtid.msctrustgate.com/NpraAuth` from `bpfk.gov.my`. Chrome's speculative preconnect confirmed via WARN log (empty header after TLS). Fixed OPTIONS regex in `WebServer.cs` (`^OPTIONS\s+(\S+)\s+HTTP`, any HTTP version) + added warn logging. Created iframe+postMessage architecture: `auth.html` served by MTID Desktop from exe web root (`bin\Debug\`), loads WebSocket from same origin (`mtid.msctrustgate.com→mtid.msctrustgate.com` = local→local, no PNA). Updated `auth.html` to read 5 params from URL query string (`UserID`, `TokenPIN`, `UserFullName`, `UserCompID`, `UserCompName`) and pass to WebSocket. Updated `bpfk.jsp` to load hidden iframe with params encoded in src URL. Copied `auth.html` to `bin\Debug\`. Pending: bpfk server JSP deployment + installer inclusion of auth.html.
- **Time Spent**: ~2 hours

### 2026-05-12 - FaultException Investigation
- **Changes**: Investigated `FaultException: Version can not be null` from `AutoUpdate.VerseCheck()`. Traced call chain to `callverchecker(vnumber, projectName)`. Config correct — user confirmed own error, no code changes.
- **Time Spent**: ~15 min

### 2026-05-08 - AboutViewModel Pattern Review
- **Changes**: Reviewed `AboutViewModel` constructor — confirmed `Task.Run()` fire-and-forget pattern correct. Minor gap: `Completed()` `MessageBox.Show` missing `DefaultDesktopOnly` (low risk). Pattern documented.
- **Time Spent**: ~15 min

### 2026-05-08 - UI Freeze Fix Session 2 (LoginViewModel + SoftCertViewModel)
- **Changes**: Fixed "Not Responding" on `LoginViewModel` and `SoftCertViewModel` — made `async Task`, wrapped blocking calls in `await Task.Run()`, added `_isProcessing` + `CanCancel` + `RelayCommand canExecute` + `Mouse.OverrideCursor` try/finally.
- **Time Spent**: ~30 min

### 2026-05-08 - UI Freeze Fix Session 1 (SelectStorageViewModel + PickupNewCertViewModel)
- **Changes**: Fixed "Not Responding" hang on both ViewModels — `async Task`, `await Task.Run()` for all blocking calls, `_isProcessing` guard, `RelayCommand canExecute`. Created `async-wpf-patterns` Jessy skill.
- **Time Spent**: ~45 min

## Historical Summary
Earlier sessions (2026-03-31 to 2026-04-21): Project registered. Full solution explored — WebSockets external repo at `C:\repos\WebSocketslog4net\WebSockets\`. Admin testing completed, BNM cert pickup success. May 6-7: STA thread fix, NullRef fix (`string.Equals`), expired cert fallback loop, CertVerifierWSClient integrated, NPRA 5-param auth (UserCompID/UserCompName), Java AES/CBC/NoPadding decrypt, `userSERIALNUMBER` overwrite bug fixed, BPFKCert single-pass parse fix, 10 error codes audited.

## Technical Notes
- **Repository**: TBD
- **Key Dependencies**: .NET Framework 4.8, WPF, WebSocket library

---
**Last Updated**: 2026-05-14 (Session 8) | **Position**: #1/10 Active
