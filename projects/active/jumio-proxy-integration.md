# jumio-proxy-integration - Adacash–Trustgate Jumio Proxy Service
*Spring Boot proxy layer between Adacash and Jumio for eKYC integration*

## Project Overview
- **Type**: Web Service / Proxy API (Spring Boot)
- **Client**: Adacash Sdn Bhd
- **Period**: 2026-04-16 - Active
- **Tech Stack**: Java 17 + Spring Boot 3.0.9 + OpenFeign + Maven + Docker + Kubernetes
- **Completion**: 70%
- **Duration**: ~5 hours
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
- **Last Session**: 2026-06-05 - Multi-project support — full implementation done, 404 issue pending
- **Next Steps**:
  1. Resolve 404 issue — understand nginx path forwarding, adjust controller paths to match
  2. Build new JAR + new Docker image with updated code
  3. Deploy new image to pilot and test end-to-end
  4. Update production deployment.yaml when pilot is confirmed
- **Known Issues**:
  - 404 on `/jumioproxy_pilot/adacash/api/v1/jumio/session` — nginx path forwarding vs Spring controller path mismatch. Context path reverted to `/adacash-jumio-proxy` but controller now has `{projectId}` in path which nginx doesn't forward. Need to clarify nginx rewrite rules before finalizing controller path structure.
  - deployment.yaml reverted to original env var format (no ConfigMap) — ConfigMap approach deferred

## Session History (Last 5)

### 2026-06-05 - Multi-Project Support Implementation
- **Changes**: Full code implementation for path-based tenant routing. New classes: `ProjectJumioProperties`, `ProjectMtssProperties`, `ProjectProperties`, `ProjectsConfigProperties`, `ProjectRegistry`, `AppConfig` (RestTemplate bean). Modified: `JumioClient` (per-project token cache via `ConcurrentHashMap`), `MtssService`/`MtssServiceImpl` (accept `ProjectMtssProperties` per call), `JumioSessionController` + `JumioCallbackController` + `JumioRetrievalController` (add `{projectId}` path variable), `AdacashTrustgateJumioProxyApplication` (register `ProjectsConfigProperties`). Deleted: `AdacashCallbackFeignClient` (replaced by RestTemplate for dynamic URL), `MtssProperties` (replaced by `ProjectMtssProperties`). Design doc written: `docs/specs/2026-06-04-multi-project-support-design.md`. Updated `application.yml` with `projects:` map. Attempted ConfigMap approach in `deployment.yaml` — reverted by user, back to original env var format. 404 issue hit after deploy — context path + nginx path mismatch, not yet resolved.
- **Time Spent**: ~3 hours

### 2026-05-11 - SESSION_EXPIRED Bug Fix
- **Changes**: Fixed `JumioCallbackController` — non-PROCESSED statuses (SESSION_EXPIRED etc.) were not forwarded to Adacash. Extracted `forwardToAdacash()` method; now ALL statuses forward to Adacash, MTSS SOAP only runs for PROCESSED. Built and deployed to pilot.
- **Time Spent**: ~20 min

### 2026-04-22 - Production Review & Archive
- **Changes**: Reviewed deployment-prod.yaml vs pilot yaml, identified 5 env vars needing production values. Same .jar/image confirmed reusable across environments.
- **Time Spent**: ~15 min

### 2026-04-21 - Image Compression Implementation
- **Changes**: Added ImageCompressionUtil + Scalr, wired into callback controller flow before MTSS SOAP call.
- **Time Spent**: ~1 session

### 2026-04-17 - Implementation Complete
- **Changes**: Full Trustgate integration — JumioCallbackController reworked, MTSS SOAP wired, images download, JAX-WS stubs generated.
- **Time Spent**: ~1 session

## Historical Summary
Project started 2026-04-16 as Trustgate proxy layer between Adacash and Jumio eKYC API. Key milestones: full callback flow implemented (MTSS SOAP, image download, Adacash forward), image compression added before SOAP call, SESSION_EXPIRED bug fixed. Pilot deployed and tested. Reactivated 2026-06-04 for multi-project (multi-tenant) support — path-based tenant routing designed and coded. Production deployment pending after 404 nginx issue resolved.

## Technical Notes
- **Repository**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\app\jumio-proxy\`
- **Artifact ID**: `adacash-trustgate-jumio-proxy` v0.0.1
- **Pilot yaml**: `pilot/deployment.yaml` — image `jumioproxy-pilot:1.07`, nodePort 30241
- **Prod yaml**: `production/deployment-prod.yaml` — image `jumioproxy:1.0`, nodePort 30240
- **Deploy command**: `kubectl apply -f deployment.yaml -n jumioproxy-pilot`
- **Rollout restart**: `kubectl rollout restart deployment/jumioproxy-pilot -n jumioproxy-pilot`
- **Key Dependencies**: Spring Boot 3.0.9, OpenFeign 4.0.3, Lombok, Commons Lang3
- **SOAP Integration**: MTSS CallbackJumio API (downstream)
- **Auth**: Jumio Basic Auth → OAuth 2.0 Bearer token flow
- **Nginx**: location `/jumioproxy_pilot` proxies to K8s NodePort 30241. Exact rewrite rule unknown — need to check before finalizing controller path structure.
- **Design doc**: `app/jumio-proxy/docs/specs/2026-06-04-multi-project-support-design.md`

---
**Last Updated**: 2026-06-05 | **Position**: #1/10 Active
