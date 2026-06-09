# jumio-proxy-integration - Adacash–Trustgate Jumio Proxy Service
*Spring Boot proxy layer between Adacash and Jumio for eKYC integration*

## Project Overview
- **Type**: Web Service / Proxy API (Spring Boot)
- **Client**: Adacash Sdn Bhd
- **Period**: 2026-04-16 - Active
- **Tech Stack**: Java 17 + Spring Boot 3.0.9 + OpenFeign + Maven + Docker + Kubernetes
- **Completion**: 90%
- **Duration**: ~10 hours
- **Due Date**: TBD

## Project Context
- **Client**: Adacash Sdn Bhd
- **Our Company**: Trustgate Sdn Bhd
- **Integration**: Trustgate acts as proxy between Adacash and Jumio eKYC API

## Flow Summary
1. Adacash routes eKYC requests through Trustgate proxy
2. Proxy returns Jumio landing page URL to user
3. User completes eKYC on Jumio side
4. Jumio sends callback to Trustgate callback URL
5. On `PROCESSED` status: retrieve workflow, download images as Base64, call MTSS SOAP, forward to Adacash
6. On non-`PROCESSED` status (e.g. `SESSION_EXPIRED`): skip MTSS, forward to Adacash directly

## Current Status
- **Last Session**: 2026-06-10 - pilot-v2 deployed, API working, pending Jumio portal callback registration
- **Next Steps**:
  1. Register new callback URL in Jumio portal (workflow key 10164): `https://digitalid.msctrustgate.com/jumioproxy_pilotv2/adacash/api/v1/jumio/callback`
  2. Full end-to-end test — trigger eKYC session, complete on Jumio, verify callback → MTSS SOAP → Adacash forward
  3. Once confirmed, merge `feature/multitenant-support` → master
  4. Update live pilot to use new multi-tenant image + nginx regex location
- **Known Issues**:
  - Jumio portal callback URL not yet registered for pilot-v2 workflow — pending Jumio portal access
  - Production URL prefix for nginx not yet confirmed (assumed `/jumioproxy`)

## Session History (Last 5)

### 2026-06-10 - pilot-v2 deployed, API working
- **Changes**: Built JAR from `feature/multitenant-support`, docker build + push to registry. Applied ConfigMap + Deployment to K8s namespace `jumioproxy-pilot`. Resolved NodePort 30242 conflict (killed old namespace). Added `namespace: jumioproxy-pilot` explicitly to all yaml files. Removed `config/application.yml` (kept only `configmap.yaml` as single source of truth). Updated nginx — added `jumioproxyPilotV2` upstream (NodePort 30242) + new regex location `/jumioproxy_pilotv2/`. Updated callback URL to `/jumioproxy_pilotv2/adacash/api/v1/jumio/callback`. Session API endpoint confirmed working ✅. Pending: Jumio portal callback registration + E2E test.
- **Time Spent**: ~1 hour

### 2026-06-09 (Session 2) - Git cleanup: master restored to clean single-tenant
- **Changes**: Manually rewrote all single-tenant versions — controllers (no `{projectId}`), `JumioClient` (single token cache), `MtssServiceImpl` (`@Value` injected), `application.yml` (flat config). Deleted multi-tenant config classes. Pushed clean master. Fixed `feature/multitenant-support` upstream tracking.
- **Time Spent**: ~1.5 hours

### 2026-06-09 (Session 1) - Multi-tenant architecture finalized + pilot-v2 setup
- **Changes**: Diagnosed 404 root cause — nginx consumes `adacash` before forwarding. Fixed with regex nginx rewrite. Renamed context path to `/jumio-proxy`. Created `pilot-v2/` self-contained folder with `configmap.yaml`, `deployment.yaml`, `Dockerfile`, nginx configs. Cleaned git branches.
- **Time Spent**: ~2.5 hours

### 2026-06-05 - Multi-Project Support Implementation
- **Changes**: Full code for path-based tenant routing. New config classes, modified all 3 controllers with `{projectId}`, `JumioClient` per-project token cache, `MtssServiceImpl` per-project props. Design doc written. 404 nginx mismatch hit.
- **Time Spent**: ~3 hours

### 2026-05-11 - SESSION_EXPIRED Bug Fix
- **Changes**: Fixed `JumioCallbackController` — non-PROCESSED statuses not forwarded to Adacash. MTSS SOAP only on PROCESSED. Deployed to pilot.
- **Time Spent**: ~20 min

## Historical Summary
Project started 2026-04-16 as Trustgate proxy layer between Adacash and Jumio eKYC API. Key milestones: full callback flow implemented (MTSS SOAP, image download, Adacash forward), image compression added, SESSION_EXPIRED bug fixed. Reactivated 2026-06-04 for multi-tenant support — path-based routing designed and coded. Master restored to clean single-tenant. pilot-v2 deployed to K8s (namespace `jumioproxy-pilot`, NodePort 30242), nginx updated. Session API confirmed working. Pending E2E test after Jumio portal callback URL registration.

## Technical Notes
- **Repository**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\app\jumio-proxy\`
- **Artifact ID**: `adacash-trustgate-jumio-proxy` v0.0.1
- **Git branches**: `master` (single-tenant, MTSS working), `feature/multitenant-support` (full multi-tenant code)
- **pilot-v2 folder**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\pilot-v2\`
- **Live pilot**: NodePort 30241 — image `jumioproxy-pilot:1.07`, context path `/adacash-jumio-proxy`
- **pilot-v2**: NodePort 30242 — image `jumioproxy-pilot-v2:1.01`, context path `/jumio-proxy`, namespace `jumioproxy-pilot`
- **Prod yaml**: `production/deployment-prod.yaml` — NodePort 30240
- **Nginx pilot-v2**: `upstream jumioproxyPilotV2 { server 127.0.0.1:30242; }` + location `~ ^/jumioproxy_pilotv2/([^/]+)/api/v1/(.*)$`
- **ConfigMap**: `pilot-v2/configmap.yaml` — single source of truth, no `config/application.yml`
- **Callback URL (pilot-v2)**: `https://digitalid.msctrustgate.com/jumioproxy_pilotv2/adacash/api/v1/jumio/callback`
- **Test endpoint**: `POST https://digitalid.msctrustgate.com/jumioproxy_pilotv2/adacash/api/v1/jumio/session`

---
**Last Updated**: 2026-06-10 | **Position**: #1/10 Active
