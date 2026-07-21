# MyTrustID Desktop - WPF Desktop App for Token/SoftCert Login & Signing
*WPF .NET Framework 4.8 desktop application with WebSocket server integration*

## Project Overview
- **Type**: Desktop Application (WPF)
- **Period**: 2026-03-31 - Active (reactivated 2026-07-09)
- **Tech Stack**: C# + WPF + .NET Framework 4.8 + WebSocket
- **Completion**: 100% (maintenance)
- **Due Date**: TBD
- **Duration**: ~16.67 hours

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
- **Last Session**: 2026-07-22 (Session 19) - Captured and analyzed the RSA-keygen/token-read crash dump via WinDbg — root cause narrowed to a `WinSCard.dll` smart-card minidriver version difference under WOW64 (32-bit process). **Decision**: hold off changing `Prefer32Bit` — QUEST3PLUS still actively used, field minidriver versions across MyTrustID token users unknown, so treating this as a targeted driver problem (fix/pin the specific minidriver) rather than an app-wide bitness change, until the rollback test proves otherwise.
- **Next Steps**:
  1. Confirm minidriver-version causality — roll failing laptop's minidriver back to `2.0.17.107` or update working laptop's up to `2.0.17.503` (see To-Do List)
  2. Report to Longmai (token vendor) if confirmed as a minidriver regression
  3. **Do not change `Prefer32Bit` yet** — only reconsider the split-build/process-isolation architecture if the minidriver rollback test fails to fix it, or if this recurs across multiple customers (see Key Logic Notes for the full decision reasoning)
  4. Resolve `.vdproj` Debug/Release packaging question — rebuild `MyTrustID` installer project with Solution Configuration = Release, then `git diff MyTrustID.vdproj` to see if `SourcePath` flips from `obj\Debug\...` to `obj\Release\...` (confirms whether the installer has ever shipped Release output)
  5. Merge `fix/autoupdate-elevation` → master (August 2026 — hold until bpfk team done)
- **Decided**: `AUT103` UserCompName-vs-certificate check removed entirely per Dejul's request (2026-07-09) — only `AUT102` (UserCompID) company check remains active
- **Known Issues**: bpfk outer iframe missing `allow="local-network-access"` — confirmed via Sec-Fetch-Dest + empty MTID header log

## Key Logic Notes
- **`AUT100` root causes found 2026-07-13**: (1) `X509Certificate2.Subject` quotes RDN values containing a comma (e.g. `CN="PAN, CHIA-YUEH"`) — naive `Split(',')` in `CertHelper.cs` tore the value in two, corrupting `CD.UserFullName`. (2) Caller (NPRA/PKI webservice) sometimes sends untrimmed whitespace in request params (e.g. `UserFullname: "Siow Nget Kam "`) — cert-side value is trimmed during DN parsing but request-side wasn't, so `OrdinalIgnoreCase` comparison still failed on the trailing space.
- `CheckBNMcert()` in `PickupNewCertViewModel.cs:507` — reads X509 SubjectDN, checks `O=BNM`
- Returns `isModifiedCka = true` if BNM cert detected → affects how cert is imported via `ImportCertificate()`
- Chrome LNA silent block signature: TLS opens, 22-sec gap, empty `Header start:` in MTID log, connection closes
- **NPRA auth (`AuthServiceNpra.cs`)**: WebSocket endpoint `/NpraAuth`, DTO `RequestAuthNpra` (`UserId`, `TokenPin`, `UserFullname`, `UserCompID`, `UserCompName`). Cert fields parsed in `CertHelper.cs` (`cd.UserOrg` = `O=`, `cd.UserOU` = `OU=Company ID - ...`). Checks: `AUT100` (UserID/UserFullName), `AUT102` (UserCompID vs `cd.UserOU`). `AUT103` (UserCompName vs `cd.UserOrg`) removed 2026-07-09.
- **Debug vs Release build difference**: only meaningful diff was `Prefer32Bit` (Debug=true, Release=false). No `#if DEBUG` blocks anywhere in codebase. PKCS#11 token driver (QUEST3PLUS) is 32-bit only — Release running AnyCPU/x64 couldn't load it, causing "token not detected". Fixed by setting `Prefer32Bit=true` in Release `PropertyGroup` in `MyTrustIDv1.csproj` (kept `Optimize=true`, `DebugType=pdbonly` — proper optimized build, not a Debug build shipped as Release).
- **`MyTrustIDv1.sln` config mapping (found 2026-07-22)**: the `MyTrustID` installer project (GUID `665F0EA7-...`) has an `ActiveCfg` but **no `Build.0`** entry for `Debug|Any CPU` — meaning the installer never builds at all under that (most-common-default) solution configuration. It only builds under `Release|AnyCPU/x64/x86` or `Debug|x64/x86`. Strong indirect evidence that Dejul has always built installers with Release active, which would mean the `.vdproj`'s cached `SourcePath` text (`obj\Debug\...`) is stale display text rather than what's actually packaged — not yet empirically confirmed (see Next Steps #4).
- **Why other token types (ST3 Ace, SafeNet) aren't affected by the `Prefer32Bit`/`WinSCard.dll` crash**: the token vendor's diagnostic tool shows MyTrustID's token (`mToken CryptoID`/Longmai) reports `MS Smart Card CSP: Installed` and a `Mini Driver Version` — meaning it's a genuine PC/SC smart card built on Microsoft's Base CSP/Minidriver framework, routing through `WinSCard.dll`/`SCardSvr`. ST3 Ace (`st3ace.dll`) and SafeNet (`eToken.dll`) are classic USB PKCS#11 dongles with their own proprietary vendor communication stacks — they likely never touch `WinSCard.dll` at all, so they're not exposed to this bug regardless of process bitness. `DetectToken.cs` treats all three identically at the C# level; the difference is entirely inside each vendor's native DLL. Not yet empirically verified (e.g. via `dumpbin /imports` on `st3ace.dll`/`eToken.dll`).
- **Decision (2026-07-22): holding off on changing `Prefer32Bit`.** QUEST3PLUS is still actively used by customers, and field minidriver versions across MyTrustID token users are unknown/uncontrolled — flipping the flag would trade one customer segment's crash for another's total failure. Since the only confirmed variable so far is a specific minidriver revision (not an inherent "32-bit + MyTrustID token" incompatibility — Dejul's own laptop runs the same 32-bit config fine), treating this as a targeted driver-version problem (fix/pin on affected machines via Longmai or IT policy) rather than an app-wide architecture change, unless the rollback test disproves it or this recurs broadly.

## Session History (Last 5)

### 2026-07-22 - RSA-Keygen Crash: Dump Captured + Root-Caused via WinDbg
- **Changes**: Dejul reproduced the crash on a second ("failing") laptop using the matching-hardware token (`mToken CryptoID`, Longmai, serial `D27ED54A37C8433E`) — crash occurred right after token detection (`SelectStorageViewModel` log line), before PIN entry, i.e. during `LoginViewModel`'s token-read flow, not necessarily during `GenerateCSR`'s keygen call as originally assumed.
- **Code-diff investigation**: `DetectToken.cs` is byte-identical between the "stable" `fix/npra-auth-async-vm` (v1.2) branch and current `fix/rsa-keygen-crash-handling` (v1.3.1) — ruled out a code-logic regression. Traced the only meaningful difference to commit `e656f8e` (2026-07-14, sessions 15-16): `Prefer32Bit` flipped `false→true` for the `Release|AnyCPU` build config, done deliberately as a fix for the **QUEST3PLUS** token driver (32-bit-only, couldn't load under a native-64-bit Release build). Also found `Debug|AnyCPU` has *always* been `Prefer32Bit=true`, on both branches — so Dejul's own dev-machine (Debug/F5) testing has always run 32-bit without issue.
- **Installer packaging caveat found**: `MyTrustID.vdproj`'s `ProjectOutput.SourcePath` is hardcoded to `..\MyTrustIDv1\obj\Debug\MyTrustIDv1.exe` on *both* branches (unchanged by `e656f8e`) — genuinely unclear whether VS Installer Projects dynamically re-resolves this to the active Solution Configuration at build time or literally always packages Debug output; not yet resolved with Dejul.
- **Dump analysis (WinDbg)**: Installed WinDbg Preview via `winget install Microsoft.WinDbg` (no debugger was present on this machine); ran `cdb.exe` (bundled at `...\Microsoft.WinDbg_.../amd64/cdb.exe`) against the `.dmp` with `!analyze -v` and `~*k` (all-thread stacks). Findings:
  - `Failure.ProblemClass.Primary: BITNESS_MISMATCH_X86_INVALID_EXCEPTION_HANDLER`, `ExceptionCode: 0xc00001a5` (`STATUS_INVALID_EXCEPTION_HANDLER` — SEH-chain corruption detected by `ntdll`, happens below the CLR entirely, which is why no C# handler — including the `[HandleProcessCorruptedStateExceptions]` guard added in session 18 — ever caught it).
  - Confirmed process ran as **x86/WOW64** (`CLR.BitnessMismatch: x86`, `OSPLATFORM_TYPE: x86`).
  - Crashing thread's full stack (thread `343c.9ac`, the only thread with WER-related frames — all 28 others were idle) shows the call chain running through `<Unloaded_mytrustid_pkcs11.dll>` → `<Unloaded_WinSCard.dll>` → SEH corruption → `ntdll!RtlDispatchException`/`RtlIsValidHandler` → `WerpWaitForCrashReporting`. WinDbg flagged frames past the `WinSCard.dll` boundary as possibly-unreliable ("Frame IP not in any known module") — expected, since stack corruption itself breaks clean unwinding; the module identities are still trustworthy.
  - Could not pull `mytrustid_pkcs11.dll`/`WinSCard.dll` version info *from the dump* — both modules were already unloaded by capture time, and unloaded-module entries only retain base address/size, not PE version resources.
- **On-machine comparison (the actual breakthrough)**: `Get-Item ....VersionInfo` on both laptops showed `mytrustid_pkcs11.dll` and `WinSCard.dll` are byte-identical versions on both machines — ruled out a stale/missing driver file theory. But the token vendor's own "Token Manager" diagnostic tool revealed the real differentiator: **same physical token** (identical serial number, firmware 3.11) but **different Smart Card Mini Driver version** — `2.0.17.107` on Dejul's (working) laptop vs. `2.0.17.503` on the failing laptop. The mini driver is a separate OS component (likely Windows-Update-delivered) that `WinSCard.dll` loads to talk to this specific card model — lines up exactly with the crash's `WinSCard.dll` boundary.
- **Conclusion**: Root cause is very likely a `mytrustid_pkcs11.dll` ↔ `WinSCard.dll` ↔ Smart Card Mini Driver interaction that only breaks under WOW64 (32-bit process, from `Prefer32Bit=true`) on mini driver `2.0.17.503` specifically. Not yet proven causally (would need to roll one machine's mini driver version to match the other and retest) — logged as the next to-do item.
- **Time Spent**: ~2 hours

### 2026-07-14 - Release Installer Prefer32Bit Retest Confirmed
- **Changes**: Tester rebuilt the Release installer with the `Prefer32Bit=true` fix (from session 15) and retested token detection — confirmed working. Closes the last open item from the token-detection bug chain.
- **Time Spent**: ~10 min

### 2026-07-13 - AUT100 Double Root-Cause Fix: Quoted-Comma DN Parsing + Untrimmed Request Params
- **Changes**: Tester retested token detection fix (session 15) — confirmed working, but surfaced two new `AUT100` (UserID/UserFullName mismatch) failures. (1) BPFK user `PAN, CHIA-YUEH`: `CertHelper.cs` parsers (`GPKICert`, `ECourtCert`, `MykeyCert`, `BPFKCert`) all used naive `subjectdn.Split(',')`, which breaks when `X509Certificate2.Subject` quotes a comma-containing RDN value (e.g. `CN="PAN, CHIA-YUEH"` → parsed as `"PAN`, truncated). Added a shared `SplitDnComponents()` helper that respects double-quoted values (and unescapes doubled `""`), applied to all four parser methods. (2) BPFK user `Siow Nget Kam`: request payload had trailing whitespace (`"Siow Nget Kam "`) that survived into the `OrdinalIgnoreCase` comparison against the (already-trimmed) cert-side value. Added `?.Trim()` to `UserID`, `UserFullName`, `UserCompID`, `UserCompName` at read-time in `AuthServiceNpra.cs` (left `TokenPIN` untouched — feeds decryption, not string comparison). Both fixes diagnosed from tester-provided debug payload logs (added last session). Not yet retested or committed.
- **Time Spent**: ~40 min

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

## Historical Summary
Earlier sessions (2026-03-31 to 2026-04-21): Project registered. Full solution explored. Admin testing completed, BNM cert pickup success. May 6-7: STA thread fix, NullRef fix, expired cert fallback loop, CertVerifierWSClient integrated, NPRA 5-param auth, Java AES/CBC/NoPadding decrypt, `userSERIALNUMBER` overwrite bug fixed, BPFKCert single-pass parse fix, 10 error codes audited. May 8: UI freeze async fix (SelectStorageViewModel + PickupNewCertViewModel). May 12: FaultException investigation — user confirmed own error, no code changes. May 14: Chrome PNA investigation — fixed `HttpService.cs` crash (query string stripping), fixed OPTIONS regex in `WebServer.cs`. May 15-17: Chrome LNA nested iframe diagnosis, JSON parse bug fix, LNA instruction box on JSPs, LNA_Findings_Draft.txt created. May 20: PickupNewCertViewModel trim fix (UserID+UUID). May 20-21: UI lock all screens (IsNotProcessing/NavEnabled pattern across 4 VMs), installer bat overhaul (mytrustid.bat rewrite), sign.bat code signing tool created. May 21-22: AutoUpdate overhaul (no admin, IExpress, restart mutex fix). Project archived 2026-06-03 (EXE+MSI signed, 100% complete) — reactivated 2026-07-09 for NPRA auth logic maintenance and a Release-build token-detection bug.

## Technical Notes
- **Repository**: `C:\repos\MyTrustIDv1_AATL-GENERIC` — sessions 15+16 committed (`e656f8e` on `fix/autoupdate-elevation`), pushed, up to date with origin
- **Key Dependencies**: .NET Framework 4.8, WPF, WebSocket library, log4net, Newtonsoft.Json
- **LNA Fix files**: `page_auth.jsp`, `page_auth_jnlp.jsp`, `page_auth_jnlp_ORI.jsp`, `error_page_auth.jsp`, `HttpService.cs`, `testq3.php`
- **SafeNet Token installer**: `C:\PROJECTS\MYTRUSTID DESKTOP\Token\Safenet\Installation` — install this before signing EXE/MSI with real code signing cert

---
**Last Updated**: 2026-07-14 (Session 17) | **Position**: #1/10 Active
