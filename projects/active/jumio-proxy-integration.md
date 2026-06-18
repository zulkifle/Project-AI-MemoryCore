# jumio-proxy-integration - Adacash–Trustgate Jumio Proxy Service
*Spring Boot proxy layer between Adacash and Jumio for eKYC integration*

## Project Overview
- **Type**: Web Service / Proxy API (Spring Boot)
- **Client**: Adacash Sdn Bhd
- **Period**: 2026-04-16 - Active
- **Tech Stack**: Java 17 + Spring Boot 3.0.9 + OpenFeign + Maven + Docker + Kubernetes
- **Completion**: 95%
- **Duration**: ~12 hours
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
- **Last Session**: 2026-06-18 - per-session callbackUrl feature + pilot v1 migration to v2 JAR + nginx actuator fix
- **Next Steps**:
  1. Apply updated nginx blocks on server (`nginx -t && nginx -s reload`)
  2. Test v1 session endpoint via Postman: `POST /jumioproxy_pilot/adacash/api/v1/jumio/session` with `callbackUrl`
  3. Full E2E test — trigger eKYC session, complete on Jumio, verify callback → MTSS SOAP → Adacash forward
  4. Merge `feature/per-session-callback-url` → `feature/multitenant-support` after E2E confirmed
- **Known Issues**: None blocking — nginx update pending on server

## Session History (Last 5)

### 2026-06-18 - per-session callbackUrl feature + v1 pilot migration
- **Changes**: Implemented `callbackUrl` feature on `feature/per-session-callback-url` branch — new `SessionCallbackStore` (ConcurrentHashMap), added optional `callbackUrl` to `CreateSessionRequest`, `resolveCallbackUrl()` in callback controller (falls back to ConfigMap URL). Tested on v2 — confirmed working. Migrated pilot v1 to use v2 JAR (multi-tenant + callbackUrl): updated `pilot/deployment.yaml` (image `1.08`, context `/jumio-proxy`, ConfigMap volume), `pilot/configmap.yaml` (fixed callback-base-url to `jumioproxy_pilot`). Updated nginx `nginx-multi-tenant.conf` for both pilot and pilot-v2 — combined API + actuator into single location block with dual rewrite directives.
- **Time Spent**: ~2 hours

### 2026-06-10 - pilot-v2 deployed, API working
- **Changes**: Built JAR from `feature/multitenant-support`, docker build + push to registry. Applied ConfigMap + Deployment to K8s namespace `jumioproxy-pilot`. Resolved NodePort 30242 conflict. Added `namespace: jumioproxy-pilot` to all yaml files. Removed `config/application.yml` (ConfigMap as single source of truth). Updated nginx — added `jumioproxyPilotV2` upstream (NodePort 30242) + regex location `/jumioproxy_pilotv2/`. Session API confirmed working ✅.
- **Time Spent**: ~1 hour

### 2026-06-09 (Session 2) - Git cleanup: master restored to clean single-tenant
- **Changes**: Manually rewrote all single-tenant versions. Pushed clean master. Fixed `feature/multitenant-support` upstream tracking.
- **Time Spent**: ~1.5 hours

### 2026-06-09 (Session 1) - Multi-tenant architecture finalized + pilot-v2 setup
- **Changes**: Diagnosed 404 root cause — nginx consumes `adacash` before forwarding. Fixed with regex nginx rewrite. Renamed context path to `/jumio-proxy`. Created `pilot-v2/` self-contained folder.
- **Time Spent**: ~2.5 hours

### 2026-06-05 - Multi-Project Support Implementation
- **Changes**: Full code for path-based tenant routing. New config classes, modified all 3 controllers with `{projectId}`, `JumioClient` per-project token cache, `MtssServiceImpl` per-project props.
- **Time Spent**: ~3 hours

## Historical Summary
Project started 2026-04-16 as Trustgate proxy layer between Adacash and Jumio eKYC API. Key milestones: full callback flow implemented (MTSS SOAP, image download, Adacash forward), image compression added, SESSION_EXPIRED bug fixed. Reactivated 2026-06-04 for multi-tenant support — path-based routing designed and coded. Master restored to clean single-tenant. pilot-v2 deployed to K8s (namespace `jumioproxy-pilot`, NodePort 30242). Per-session callbackUrl feature implemented and tested. pilot v1 migrated to use v2 multi-tenant JAR.

## Technical Notes
- **Repository**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\app\jumio-proxy\`
- **Artifact ID**: `adacash-trustgate-jumio-proxy` v0.0.1
- **Git branches**: `master` (single-tenant), `feature/multitenant-support` (multi-tenant), `feature/per-session-callback-url` (callbackUrl feature — based on multitenant)
- **pilot folder**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\pilot\` — NodePort 30241, image `jumioproxy-pilot:1.08`, context `/jumio-proxy`
- **pilot-v2 folder**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\pilot-v2\` — NodePort 30242, image `jumioproxy-pilot-v2:1.02`, context `/jumio-proxy`
- **Prod yaml**: `production/deployment-prod.yaml` — NodePort 30240
- **Nginx**: Combined location block handles both `/api/v1/` and `/actuator/` via dual rewrite directives
- **ConfigMap**: single source of truth for per-project Jumio + MTSS + clientCallbackUrl config
- **Callback URL (pilot)**: `https://digitalid.msctrustgate.com/jumioproxy_pilot/adacash/api/v1/jumio/callback`
- **Callback URL (pilot-v2)**: `https://digitalid.msctrustgate.com/jumioproxy_pilotv2/adacash/api/v1/jumio/callback`
- **Session endpoint (pilot)**: `POST https://digitalid.msctrustgate.com/jumioproxy_pilot/adacash/api/v1/jumio/session`
- **Session endpoint (pilot-v2)**: `POST https://digitalid.msctrustgate.com/jumioproxy_pilotv2/adacash/api/v1/jumio/session`

---
**Last Updated**: 2026-06-18 | **Position**: #1/10 Active
