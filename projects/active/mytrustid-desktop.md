# MyTrustID Desktop - WPF Desktop App for Token/SoftCert Login & Signing
*WPF .NET Framework 4.8 desktop application with WebSocket server integration*

## Project Overview
- **Type**: Desktop Application (WPF)
- **Period**: 2026-03-31 - Active
- **Tech Stack**: C# + WPF + .NET Framework 4.8 + WebSocket
- **Completion**: 97%
- **Due Date**: 2026-04-15
- **Duration**: ~3.75 hours

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
- **Last Session**: 2026-05-12 - FaultException investigation (case closed — user error)
- **Next Steps**: Full end-to-end test — all flows with real token/softcert
- **Completed**: STA thread fix ✓, NullRef fix ✓, expired cert fallback ✓, CertVerifier integrated ✓, NPRA 5-param auth ✓, BPFKCert fix ✓, error codes verified ✓, UI freeze async fix (all 4 ViewModels) ✓, AboutViewModel fire-and-forget pattern reviewed ✓
- **Known Issues**: None

## Key Logic Notes
- `CheckBNMcert()` in `PickupNewCertViewModel.cs:507` — reads X509 SubjectDN, checks `O=BNM`
- Returns `isModifiedCka = true` if BNM cert detected → affects how cert is imported via `ImportCertificate()`

## Session History (Last 5)

### 2026-05-12 - FaultException Investigation
- **Changes**: Investigated `FaultException: Version can not be null` from `AutoUpdate.VerseCheck()`. Traced call chain: `VerseCheck()` → `VersionCheckerClient.VersionChecker()` → `callverchecker(vnumber, projectName)`. Checked `App.config` and `bin\Debug` config — both have `VersionNumber = 1.2`. Located installed exe at `C:\Program Files (x86)\Trustgate\MyTrustID\`. User confirmed it was their own error — case closed, no code changes.
- **Time Spent**: ~15 min

### 2026-05-08 - AboutViewModel Pattern Review
- **Changes**: Reviewed `AboutViewModel` constructor — confirmed `Task.Run(() => update.VerseCheck())` and `Task.Run(() => update.WSSCheck())` are correct fire-and-forget pattern. No UI freeze risk — both methods have root-level `try/catch` and use `MessageBoxOptions.DefaultDesktopOnly` for thread-safe dialogs. Minor gap noted: `Completed()` callback `MessageBox.Show` missing `DefaultDesktopOnly` (low risk). Pattern documented for reuse in async-wpf-patterns skill.
- **Time Spent**: ~15 min

### 2026-05-08 - UI Freeze Fix Session 2 (LoginViewModel + SoftCertViewModel)
- **Changes**: Fixed "Not Responding" on `LoginViewModel` login button — `LoadCertDetail()` → `async Task`, `readtoken.GetTokenCert()` → `await Task.Run()`, added `_isProcessing` + `CanCancel` + `RelayCommand canExecute` + `Mouse.OverrideCursor` try/finally. Fixed same on `SoftCertViewModel` OK button — `LoadCertDetail()` → `async Task`, both `readSCert.GetSoftCert()` branches wrapped in `await Task.Run()`, added `CanLoad` + `CanCancel` + `RelayCommand canExecute`. Explained async/await mechanics to user (how await waits on that line, not at end of method).
- **Time Spent**: ~30 min

### 2026-05-08 - UI Freeze Fix Session 1 (SelectStorageViewModel + PickupNewCertViewModel)
- **Changes**: Fixed "Not Responding" hang on `SelectStorageViewModel.LoadTokenType()` — made `async Task`, wrapped `DetectToken.Token()` in `await Task.Run()`, added `_isProcessing` + `CanLoadTokenType` + `CanCancel` + `Mouse.OverrideCursor` with `try/finally`. Fixed same issue on `PickupNewCertViewModel.PickupCert()` — made `async Task`, wrapped 5 blocking calls in `await Task.Run()` (token detect, PKCS#11 init+CSR gen grouped in single Task.Run for thread safety, HTTP call, cert import, cert read), updated `RelayCommand` with `canExecute: _ => !_isProcessing`, removed `StackedCursorOverride`. Created `async-wpf-patterns` Jessy skill capturing the pattern for future use.
- **Time Spent**: ~45 min

### 2026-05-07 - BPFKCert Fix Confirmed + Error Code Audit
- **Changes**: Confirmed single-pass BPFKCert parsing fix resolves AUT100 with correct params — `OU = MyKad / Passport No – 840622015795` now correctly parsed into `userSERIALNUMBER`. Audited all error paths in `AuthServiceNpra.cs` — verified 10 error codes are fully covered with no gaps (AUT3001, 100, ReadTokenCert passthrough, AUT100, AUT102, AUT103, -203, -301, other cert codes, AUT101).
- **Time Spent**: ~15 min

### 2026-05-06 - NPRA Auth Enhancement (UserCompID/UserCompName + Java AES)
- **Changes**: Added `UserCompID` and `UserCompName` to `RequestAuthNpra` model. Updated `AuthServiceNpra` to deserialize, validate, and compare all 5 params against cert SubjectDN (`UserOU`→UserCompID, `UserOrg`→UserCompName). Fixed `userSERIALNUMBER` overwrite bug in `CertHelper.cs` (line 354-355 double assignment). Replaced old `EncryptDecryptHelper` decryption with Java-compatible AES/CBC/NoPadding (embedded IV, key `msctrustgate2015` hardcoded). Changed TokenPin log to show length only for security. Added `DecryptedTokenJava()` test method in `Tester/Program.cs`.
- **Time Spent**: ~45 min

### 2026-05-06 - NPRA Auth Stability (NullRef + Expired Cert + CertVerifier)
- **Changes**: Fixed `NullReferenceException` in `AuthServiceNpra` — `cd.UserSN` was null for BPFK certs, replaced `.Equals()` with `string.Equals()`. Fixed expired cert condition in `ReadTokenCert.cs:586` — changed `(CAFlag == false && certcount == 1) || certStatus == "Valid"` to `certStatus == "Valid"` only. Added expired cert fallback logic: loop tracks `latestExpiredCD`, writes it to XML after loop if no valid cert found. Integrated `CertVerifierWSClient` into NPRA auth flow — cert validated via SOAP before UserID comparison, blocks on revoked/expired/invalid.
- **Time Spent**: ~60 min

## Historical Summary
Earlier sessions (2026-03-31 to 2026-04-21): Project registered into LRU system. Explored full solution structure — confirmed WebSockets external repo at `C:\repos\WebSocketslog4net\WebSockets\`, identified Server\ folder with service classes. Admin testing completed — BNM cert pickup success. STA thread fix, NullRef fix, expired cert fallback, NPRA 5-param auth, BPFKCert single-pass parse fix all delivered across May 6-7 sessions.

## Technical Notes
- **Repository**: TBD
- **Key Dependencies**: .NET Framework 4.8, WPF, WebSocket library

---
**Last Updated**: 2026-05-08 (Session 7) | **Position**: #1/10 Active
