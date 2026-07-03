# jumio-proxy-integration - Adacash–Trustgate Jumio Proxy Service
*Spring Boot proxy layer between Adacash and Jumio for eKYC integration*

## Project Overview
- **Type**: Web Service / Proxy API (Spring Boot)
- **Client**: Adacash Sdn Bhd
- **Period**: 2026-04-16 - 2026-06-26
- **Tech Stack**: Java 17 + Spring Boot 3.0.9 + OpenFeign + Maven + Docker + Kubernetes
- **Completion**: 100%
- **Duration**: ~14 hours
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
- **Last Session**: 2026-06-26 - PROD deployed, image pushed, K8s applied, nginx updated, Adacash tested ✅
- **Remaining**: Merge `feature/per-session-callback-url` → `feature/multitenant-support` after PROD confirmed
- **Known Issues**: None

## Session History (Last 5)

### 2026-06-26 - PROD Deployed & Tested ✅
- **Changes**: Docker image built and pushed (`localhost:30445/jumioproxy:1.0`). Namespace `jumioproxy` created. `configmap.yaml` + `deployment.yaml` applied to K8s. Nginx updated with `jumioproxyProd` upstream (port 30240) + location block, reloaded. Adacash re-tested on PROD URL — confirmed working end-to-end.
- **Time Spent**: ~30 min

### 2026-06-23 - PROD deployment YAML prep
- **Changes**: Prepared full production deployment package — rewrote `production/deployment.yaml`, `production/configmap.yaml`, `production/nginx/nginx-prod.conf`.
- **Time Spent**: ~45 min

### 2026-06-19 - callbackUrl debug + imagePullPolicy fix
- **Changes**: Fixed `imagePullPolicy: IfNotPresent` — K8s used cached old image. Fixed: set `imagePullPolicy: Always`, cycled tag 1.07→1.08. Both v1 and v2 confirmed working end-to-end.
- **Time Spent**: ~45 min

### 2026-06-18 - per-session callbackUrl feature + v1 pilot migration
- **Changes**: Implemented `SessionCallbackStore`, added optional `callbackUrl` to `CreateSessionRequest`. Migrated pilot v1 to v2 JAR.
- **Time Spent**: ~2 hours

### 2026-06-10 - pilot-v2 deployed, API working
- **Changes**: Built JAR from `feature/multitenant-support`, docker build + push, applied ConfigMap + Deployment. NodePort 30242 confirmed working.
- **Time Spent**: ~1 hour

## Historical Summary
Project started 2026-04-16 as Trustgate proxy layer between Adacash and Jumio eKYC API. Key milestones: full callback flow implemented (MTSS SOAP, image download, Adacash forward), image compression added, SESSION_EXPIRED bug fixed. Reactivated 2026-06-04 for multi-tenant support — path-based routing designed and coded. Master restored to clean single-tenant. pilot-v2 deployed to K8s. Per-session callbackUrl feature implemented. PROD deployed and tested 2026-06-26.

## Technical Notes
- **Repository**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\app\jumio-proxy\`
- **Git branches**: `master` (single-tenant), `feature/multitenant-support`, `feature/per-session-callback-url`
- **Prod namespace**: `jumioproxy` | NodePort: 30240 | Image: `localhost:30445/jumioproxy:1.0`
- **Pilot**: NodePort 30241, image `jumioproxy-pilot:1.08`
- **Pilot-v2**: NodePort 30242, image `jumioproxy-pilot-v2:1.02`
- **Callback URL (prod)**: `https://digitalid.msctrustgate.com/jumioproxy/adacash/api/v1/jumio/callback`
- **Pending**: Merge `feature/per-session-callback-url` → `feature/multitenant-support`

---
**Last Updated**: 2026-07-03 | **Status**: Archived (LRU) — Completed ✅ 2026-06-26. PROD live, Adacash tested.
