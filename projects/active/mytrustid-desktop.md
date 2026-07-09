# MyTrustID Desktop - WPF Desktop App for Token/SoftCert Login & Signing
*WPF .NET Framework 4.8 desktop application with WebSocket server integration*

## Project Overview
- **Type**: Desktop Application (WPF)
- **Period**: 2026-03-31 - Active (reactivated 2026-07-09)
- **Tech Stack**: C# + WPF + .NET Framework 4.8 + WebSocket
- **Completion**: 100% (maintenance)
- **Due Date**: TBD
- **Duration**: ~15.75 hours

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
- **Last Session**: 2026-07-09 - NPRA company-name check removed, debug payload logging added, Release `Prefer32Bit` fix for token-detection bug
- **Next Steps**:
  1. Rebuild Release installer with `Prefer32Bit=true` fix, have tester retest token detection
  2. Confirm removal of `AUT103` (UserCompName) check doesn't break NPRA auth flow expectations on the caller (PKI webservice) side
  3. Changes not yet committed in `MyTrustIDv1_AATL-GENERIC` repo — commit when tester confirms fix
  4. Merge `fix/autoupdate-elevation` → master (August 2026 — hold until bpfk team done)
- **Decided**: `AUT103` UserCompName-vs-certificate check removed entirely per Dejul's request (2026-07-09) — only `AUT102` (UserCompID) company check remains active
- **Known Issues**: bpfk outer iframe missing `allow="local-network-access"` — confirmed via Sec-Fetch-Dest + empty MTID header log

## Key Logic Notes
- `CheckBNMcert()` in `PickupNewCertViewModel.cs:507` — reads X509 SubjectDN, checks `O=BNM`
- Returns `isModifiedCka = true` if BNM cert detected → affects how cert is imported via `ImportCertificate()`
- Chrome LNA silent block signature: TLS opens, 22-sec gap, empty `Header start:` in MTID log, connection closes
- **NPRA auth (`AuthServiceNpra.cs`)**: WebSocket endpoint `/NpraAuth`, DTO `RequestAuthNpra` (`UserId`, `TokenPin`, `UserFullname`, `UserCompID`, `UserCompName`). Cert fields parsed in `CertHelper.cs` (`cd.UserOrg` = `O=`, `cd.UserOU` = `OU=Company ID - ...`). Checks: `AUT100` (UserID/UserFullName), `AUT102` (UserCompID vs `cd.UserOU`). `AUT103` (UserCompName vs `cd.UserOrg`) removed 2026-07-09.
- **Debug vs Release build difference**: only meaningful diff was `Prefer32Bit` (Debug=true, Release=false). No `#if DEBUG` blocks anywhere in codebase. PKCS#11 token driver (QUEST3PLUS) is 32-bit only — Release running AnyCPU/x64 couldn't load it, causing "token not detected". Fixed by setting `Prefer32Bit=true` in Release `PropertyGroup` in `MyTrustIDv1.csproj` (kept `Optimize=true`, `DebugType=pdbonly` — proper optimized build, not a Debug build shipped as Release).

## Session History (Last 5)

### 2026-07-09 - NPRA Company Name Check Removal + Debug Payload Logging + Release Config Fix
- **Changes**: (1) Investigated NPRA cert-vs-param comparison logic in `AuthServiceNpra.cs` — confirmed `AUT103` compared cert `O=` (`cd.UserOrg`) against request `UserCompName`. (2) Added payload logging after deserialization (`payloadReceived` var + `log.InfoFormat("Payload received: {0}", payloadReceived)`), masking `TokenPin` — fixed SonarC# S2629 (logging template must be constant, no string concatenation). (3) Removed the `AUT103` UserCompName-vs-cert `else if` block entirely per request — `AUT102` (UserCompID) check still active, chain now falls through to cert status check. (4) Diagnosed tester-reported "token not detected" issue on compiled Release installer — traced to `Prefer32Bit=false` in Release vs `true` in Debug in `MyTrustIDv1.csproj` (PKCS#11 token driver is 32-bit only, so Release running as x64 couldn't load it). Dejul initially matched full Debug settings (`DebugType=full`, `Optimize=false`) in Release too; reverted those two back to `pdbonly`/`true` per confirmation, kept only `Prefer32Bit=true`.
- **Time Spent**: ~45 min

### 2026-05-22 - AutoUpdate Overhaul — No Admin, IExpress, Restart Fix
- **Changes**: (1) `app.manifest` reverted to `asInvoker` — root cause was runtime writes, not install. (2) `wss.txt` path moved to `C:\Trustgate\MyTrustID\wss.txt` (App.config + Service.cs). (3) `VerseCheck()`: removed admin check, fixed `using` block async bug (WebClient disposed early), launches `MyTrustID.EXE` (IExpress) via `Verb=runas` — UAC one-click. (4) `WSSCheck()`: fully silent cert download + notify popup + clean restart. (5) `Completed()`: added `e.Error`/`e.Cancelled` checks, download path moved to `C:\Trustgate\MyTrustID\`. (6) `RestartApplication()`: fixed cross-thread crash with `Dispatcher.Invoke`; fixed mutex race via `App.IsRestarting` flag — new process starts in `OnExit` after NotifyIcon disposed + mutex released. Branch: `fix/autoupdate-elevation` — tested ✅ pushed ✅.
- **Time Spent**: ~3 hours

### 2026-05-21 - app.manifest UAC Elevation (Reverted this session)
- **Changes**: Changed `requestedExecutionLevel` to `requireAdministrator` — later reverted to `asInvoker` after root cause analysis.
- **Time Spent**: ~15 min

### 2026-05-21 - page_auth.jsp LNA Instruction Box Enhancement
- **Changes**: Replaced non-functional "click here" anchor (blocked by Chrome security) with "Copy & Open New Tab" button. Button copies `chrome://` or `edge://` flags URL to clipboard and opens a blank new tab in one click. Added browser restriction explanatory text. Pending: same changes to `page_auth_jnlp.jsp` (deferred).
- **Time Spent**: ~30 min

### 2026-05-21 - UI Lock All Screens + Installer Bat Overhaul + Code Signing Tool
- **Changes**: (1) Added `IsNotProcessing` + `NavEnabled` pattern across all 4 VMs — `PickupNewCertViewModel`, `SelectStorageViewModel`, `LoginViewModel`, `SoftCertViewModel` — locks all input fields, radio buttons, and sidebar nav during processing, re-enables in `finally`. (2) Rewrote `mytrustid.bat`: replaced dynamic .vbs→PowerShell shortcut, bitsadmin→WebClient.DownloadFile, .NET 4.6.1→4.8 (Release DWORD check), WMIC→registry + direct `msiexec /x` (synchronous), fixed auto-run race condition. (3) Created `sign.bat` — double-click code signing tool for SafeNet token using `signtool /a`.
- **Time Spent**: ~2 hours

## Historical Summary
Earlier sessions (2026-03-31 to 2026-04-21): Project registered. Full solution explored. Admin testing completed, BNM cert pickup success. May 6-7: STA thread fix, NullRef fix, expired cert fallback loop, CertVerifierWSClient integrated, NPRA 5-param auth, Java AES/CBC/NoPadding decrypt, `userSERIALNUMBER` overwrite bug fixed, BPFKCert single-pass parse fix, 10 error codes audited. May 8: UI freeze async fix (SelectStorageViewModel + PickupNewCertViewModel). May 12: FaultException investigation — user confirmed own error, no code changes. May 14: Chrome PNA investigation — fixed `HttpService.cs` crash (query string stripping), fixed OPTIONS regex in `WebServer.cs`. May 15-17: Chrome LNA nested iframe diagnosis, JSON parse bug fix, LNA instruction box on JSPs, LNA_Findings_Draft.txt created. May 20: PickupNewCertViewModel trim fix (UserID+UUID). May 20-22: UI lock all screens, installer bat overhaul, code signing tool, AutoUpdate overhaul (no admin, IExpress, restart mutex fix). Project archived 2026-06-03 (EXE+MSI signed, 100% complete) — reactivated 2026-07-09 for NPRA auth logic maintenance and a Release-build token-detection bug.

## Technical Notes
- **Repository**: `C:\repos\MyTrustIDv1_AATL-GENERIC` (not yet committed — pending changes from 2026-07-09 session)
- **Key Dependencies**: .NET Framework 4.8, WPF, WebSocket library, log4net, Newtonsoft.Json
- **LNA Fix files**: `page_auth.jsp`, `page_auth_jnlp.jsp`, `page_auth_jnlp_ORI.jsp`, `error_page_auth.jsp`, `HttpService.cs`, `testq3.php`
- **SafeNet Token installer**: `C:\PROJECTS\MYTRUSTID DESKTOP\Token\Safenet\Installation` — install this before signing EXE/MSI with real code signing cert

---
**Last Updated**: 2026-07-09 (Session 15) | **Position**: #1/10 Active
