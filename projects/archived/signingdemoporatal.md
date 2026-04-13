# signingDemoPortal - Demo Portal for Client Integration & Digital Signing
*End-to-end demo showcasing user verification and digital signing flow for client integration*

## Project Overview
- **Type**: Web Application (Demo Portal)
- **Period**: 2026-04-05 - 2026-04-10
- **Tech Stack**: HTML / JavaScript (Frontend) + Java Servlet WAR (Backend) + WebSocket (SockJS/STOMP) + SOAP Web Service
- **Completion**: 90%
- **Duration**: 3 hours

## Project Path
- **Root**: `C:\PROJECTS\DEMO\MYEG-SIGNING-PORTAL\Development\DSPortalDemo\`
- **Sample HTML**: `pkiauth_pilot_ws-client.html` (existing — reuse as-is)
- **Backend**: Java Servlet WAR (Maven, Java 1.8, deployed to Tomcat)
- **Demo Page**: `src/main/webapp/demo.html` → `http://localhost:8080/SandboxAPI/DSPortalDemo/demo.html`
- **Context Path**: `/SandboxAPI/DSPortalDemo` (deployed inside mtsa-sandbox Docker)

## Objectives
Demo portal for clients to see full integration flow:
1. User Verification (WebSocket — 2 types) ✓
2. Certificate flow (SignPDFFile — enrol + sign combined) ✓
3. PDF Digital Signing (SOAP web service) ✓
4. Auto-download signed PDF ✓
5. Certificate info display (GetCertInfo) ✓

## Requirements Summary

### Frontend
- Reuse existing `pkiauth_pilot_ws-client.html` — DO NOT modify verification UI/logic
- Signing button (disabled by default, enabled after successful verification) ✓
- Status display section (Processing / Success / Failed) ✓
- On signing complete: auto-download signed PDF + show cert info ✓
- DS999 session expired handling — disable button, prompt re-auth ✓
- Certificate info table: certStatus green/bold if Valid, X.509 downloadable as .pem ✓

### Backend
- `SignPDFFile` SOAP API — enrol + sign in one call ✓
- `GetCertInfo` SOAP API — cert details after signing ✓
- JAX-WS stubs: `com.msctg.mtsaws.rasm` package ✓
- Credentials: `demorael_sandbox_everify` / `YcuLxvMMcXWPLRaW` (HTTP headers) ✓
- Dummy PDF: `sample.pdf` from classpath ✓
- Signature: centered on A4 — x1=172, x2=330, y1=381, y2=80 ✓

### Docker/K8s
- Deployed inside `mtsa-sandbox` Docker image ✓
- Context: `SandboxAPI#DSPortalDemo` ✓
- Image: `localhost:30445/sandbox-mtsa:1.062` ✓
- NodePort: `30053` ✓
- BouncyCastle JCE provider registered in Dockerfile ✓

## Current Status
- **Last Session**: 2026-04-06 - Full implementation complete. Deployed and tested successfully.
- **Next Steps**: None — demo is live and functional
- **Known Issues**: None

## Session History (Last 5)

### 2026-04-06 - Full SOAP Implementation + Docker Deployment
- **Changes**:
  - Implemented `SignPDFFile` + `GetCertInfo` via JAX-WS stubs (`com.msctg.mtsaws.rasm`)
  - Added WSDL to `pom.xml`, added `jaxws-rt 2.3.2` runtime dependency
  - Fixed `SignPDFFile_pilot` array bug (size 4, correct field mapping)
  - Added `GetCertInfo_pilot` following same pattern
  - Added HTTP credentials headers (`Username` / `Password`)
  - Loaded dummy PDF (`sample.pdf`) from classpath
  - Updated sample API key display
  - Signature appearance centered on A4 (x1=172, x2=330, y1=381, y2=80)
  - Fixed context path detection in `demo.html` for nested paths
  - Updated frontend: JSON response handling, cert info panel, DS999 session handling
  - Fixed BouncyCastle JCE provider in Dockerfile
  - Deployed as `SandboxAPI#DSPortalDemo` in mtsa-sandbox Docker (image 1.062)
  - Tested end-to-end — deployment successful
- **Time Spent**: ~3 hours

### 2026-04-06 - Frontend + Backend Scaffold
- **Changes**:
  - Created `demo.html` — verification flow reused, signing button added
  - Created `SignServlet.java` — handles POST /sign
  - Created `SigningService.java` — full cert flow with TBI placeholders
  - Created `AuthProxyServlet.java` — proxies login to avoid CORS block
  - Fixed SockJS URL (https not wss), fixed Java 8 readAllBytes issue
  - Packaging changed to WAR, servlet-api dependency added
  - Verification flow tested successfully in browser
- **Time Spent**: ~60 min

### 2026-04-05 - Project Created
- **Changes**: Project registered. Full requirements captured — frontend UI rules, backend cert flow (TBI), PDF signing, SOAP integration.
- **Time Spent**: ~0 min

## Historical Summary
[No history yet — this section is populated when session count exceeds 5]

## Technical Notes
- **Repository**: TBD
- **Key Dependencies**: javax.servlet-api 3.1.0, jaxws-rt 2.3.2, log4j 2.18.0
- **Auth Server**: `https://digitalid.msctrustgate.com/pkiauth_pilot` (external)
- **MTSA Endpoint**: `https://digitalid.msctrustgate.com/SandboxAPI/MTSASandboxRASM/MyTrustSignerAgentWSRASM?wsdl`
- **Key Files**:
  - `src/main/webapp/demo.html` — landing page
  - `com.msctg.demo.SignServlet` — POST /sign
  - `com.msctg.demo.SigningService` — SignPDFFile + GetCertInfo
  - `com.msctg.demo.AuthProxyServlet` — POST /auth/login (CORS proxy)
  - `src/main/resources/sample.pdf` — dummy PDF for signing
- **Sandbox Docker**: `C:\PROJECTS\DOCKER GITLAB\docker\mtsa-sandbox\`

---
**Last Updated**: 2026-04-10 | **Status**: Archived (Completed) - 2026-04-10
