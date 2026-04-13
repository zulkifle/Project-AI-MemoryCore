# MyTrustID Desktop - WPF Desktop App for Token/SoftCert Login & Signing
*WPF .NET Framework 4.8 desktop application with WebSocket server integration*

## Project Overview
- **Type**: Desktop Application (WPF)
- **Period**: 2026-03-31 - Active
- **Tech Stack**: C# + WPF + .NET Framework 4.8 + WebSocket
- **Completion**: 80%
- **Due Date**: 2026-04-13
- **Duration**: 0 min

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
- **Last Session**: 2026-04-02 - Explored solution and project structure
- **Next Steps**: 1) Compile new .exe via MyTrustID installer project 2) Test pickup cert flow with BNM cert
- **Known Issues**: None logged yet

## Key Logic Notes
- `CheckBNMcert()` in `PickupNewCertViewModel.cs:507` — reads X509 SubjectDN, checks `O=BNM`
- Returns `isModifiedCka = true` if BNM cert detected → affects how cert is imported via `ImportCertificate()`

## Session History (Last 5)

### 2026-04-02 - Short Session
- **Changes**: No code changes. Confirmed load command: `load project mytrustid-desktop`
- **Time Spent**: ~2 min

### 2026-04-01 - Project Exploration
- **Changes**: Explored full solution structure. Confirmed WebSockets is an external repo (`C:\repos\WebSocketslog4net\WebSockets\`) referenced in the .sln. Identified Server\ folder inside MyTrustIDv1 contains service classes (AuthService, SignHashService, SignPDFService, etc.)
- **Time Spent**: ~10 min

### 2026-03-31 - Project Created
- **Changes**: Project registered into Jessy LRU Project Management System
- **Time Spent**: ~0 min

## Historical Summary
[No history yet — this section is populated when session count exceeds 5]

## Technical Notes
- **Repository**: TBD
- **Key Dependencies**: .NET Framework 4.8, WPF, WebSocket library

---
**Last Updated**: 2026-04-02 (resumed) | **Position**: #1/10 Active
