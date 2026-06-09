# jumio-proxy-integration - Adacash–Trustgate Jumio Proxy Service
*Spring Boot proxy layer between Adacash and Jumio for eKYC integration*

## Project Overview
- **Type**: Web Service / Proxy API (Spring Boot)
- **Client**: Adacash Sdn Bhd
- **Period**: 2026-04-16 - Active
- **Tech Stack**: Java 17 + Spring Boot 3.0.9 + OpenFeign + Maven + Docker + Kubernetes
- **Completion**: 80%
- **Duration**: ~7.5 hours
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
- **Last Session**: 2026-06-09 - Multi-tenant architecture finalized, pilot-v2 ready for deployment
- **Next Steps**:
  1. Build JAR from `feature/multitenant-support` branch → place in `pilot-v2/`
  2. Docker build + push on K8s server: `docker build -t localhost:30445/jumioproxy-pilot-v2:1.01 .`
  3. Apply to K8s: `kubectl apply -f configmap.yaml -n jumioproxy-pilot && kubectl apply -f deployment.yaml -n jumioproxy-pilot`
  4. Update nginx — replace old `location /jumioproxy_pilot/adacash` with regex blocks from `pilot-v2/nginx/nginx-multi-tenant.conf`
  5. Test end-to-end with Adacash on pilot-v2 (NodePort 30242)
  6. Once confirmed, merge `feature/multitenant-support` → master, deploy to live pilot
- **Known Issues**:
  - Nginx upstream `jumioproxyPilotV2` (NodePort 30242) not yet defined — need to add upstream block before applying nginx config
  - Production URL prefix for nginx not yet confirmed (assumed `/jumioproxy`)

## Session History (Last 5)

### 2026-06-09 - Multi-tenant architecture finalized + pilot-v2 setup
- **Changes**: Diagnosed 404 root cause — nginx location `/jumioproxy_pilot/adacash` consumes the `adacash` segment before forwarding to Spring. Fixed with regex nginx rewrite that captures `{projectId}` dynamically. Renamed context path `/adacash-jumio-proxy` → `/jumio-proxy` for tenant neutrality. Created `pilot-v2/` as self-contained deployment folder: `configmap.yaml` (per-tenant config, separate from deployment), `deployment.yaml` (Deployment + Service, NodePort 30242), `Dockerfile` (copies JAR from same folder), `nginx/nginx-multi-tenant.conf` (regex location blocks for pilot + prod). Cleaned git: master reverted to pre-multi-tenant state, `feature/multitenant-support` branch created with full multi-tenant code. Pushed SESSION_EXPIRED fix (May 2026, was unpushed). Old `feature/multi-tenant-context-path` branch deleted.
- **Time Spent**: ~2.5 hours

### 2026-06-05 - Multi-Project Support Implementation
- **Changes**: Full code implementation for path-based tenant routing. New classes: `ProjectJumioProperties`, `ProjectMtssProperties`, `ProjectProperties`, `ProjectsConfigProperties`, `ProjectRegistry`, `AppConfig` (RestTemplate bean). Modified all 3 controllers with `{projectId}` path variable, `JumioClient` per-project token cache, `MtssServiceImpl` accept per-project props. Deleted `AdacashCallbackFeignClient`. Design doc written. 404 nginx path mismatch hit after deploy.
- **Time Spent**: ~3 hours

### 2026-05-11 - SESSION_EXPIRED Bug Fix
- **Changes**: Fixed `JumioCallbackController` — non-PROCESSED statuses not forwarded to Adacash. Extracted `forwardToAdacash()`; now ALL statuses forward, MTSS SOAP only on PROCESSED. Built and deployed to pilot.
- **Time Spent**: ~20 min

### 2026-04-22 - Production Review & Archive
- **Changes**: Reviewed deployment-prod.yaml vs pilot yaml, identified 5 env vars needing production values.
- **Time Spent**: ~15 min

### 2026-04-21 - Image Compression Implementation
- **Changes**: Added ImageCompressionUtil + Scalr, wired into callback controller flow before MTSS SOAP call.
- **Time Spent**: ~1 hour

## Historical Summary
Project started 2026-04-16 as Trustgate proxy layer between Adacash and Jumio eKYC API. Key milestones: full callback flow implemented (MTSS SOAP, image download, Adacash forward), image compression added, SESSION_EXPIRED bug fixed and deployed. Reactivated 2026-06-04 for multi-tenant support — path-based routing designed, coded, and git-organized. pilot-v2 deployment folder fully prepared. Awaiting JAR build + K8s deploy + nginx update to go live.

## Technical Notes
- **Repository**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\app\jumio-proxy\`
- **Artifact ID**: `adacash-trustgate-jumio-proxy` v0.0.1
- **Git branches**: `master` (stable, pre-multi-tenant), `feature/multitenant-support` (full multi-tenant code)
- **pilot-v2 folder**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\pilot-v2\`
- **Live pilot**: `pilot/deployment.yaml` — image `jumioproxy-pilot:1.07`, nodePort 30241
- **pilot-v2**: `pilot-v2/deployment.yaml` — image `jumioproxy-pilot-v2:1.01`, nodePort 30242
- **Prod yaml**: `production/deployment-prod.yaml` — image `jumioproxy:1.0`, nodePort 30240
- **Nginx fix**: regex location blocks in `pilot-v2/nginx/nginx-multi-tenant.conf` — captures `{projectId}` dynamically, no project name hardcoded
- **ConfigMap**: per-tenant config in `pilot-v2/configmap.yaml` — add tenant = edit ConfigMap + rollout restart, no rebuild
- **Spring config**: `SPRING_CONFIG_ADDITIONAL_LOCATION=file:/config/` mounts ConfigMap into pod
- **Context path**: `/jumio-proxy` (neutral, not client-specific)

---
**Last Updated**: 2026-06-09 | **Position**: #1/10 Active
