# jumio-proxy-integration - Adacash–Trustgate Jumio Proxy Service
*Spring Boot proxy layer between Adacash and Jumio for eKYC integration*

## Project Overview
- **Type**: Web Service / Proxy API (Spring Boot)
- **Client**: Adacash Sdn Bhd
- **Period**: 2026-04-16 - Active
- **Tech Stack**: Java 17 + Spring Boot 3.0.9 + OpenFeign + Maven + Docker + Kubernetes
- **Completion**: 100% (maintenance)
- **Duration**: ~16 hours
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
- **Last Session**: 2026-07-16 - Missing userId (documentNumber) now treated as failed eKYC (MTSS skipped cleanly)
- **Next Steps**:
  1. **Rebuild Docker image + redeploy to PROD** — the userId check (commit `03bf023`) is pushed but NOT yet deployed; prod still runs old image
  2. Monitor logs over next token-expiry cycles to confirm 401s no longer recur
  3. Await Jumio's confirmation after their own monitoring
  4. Merge `feature/per-session-callback-url` → `feature/multitenant-support` after fixes confirmed stable ← still remaining
- **Known Issues**: None (userId-check redeploy pending)

## Session History (Last 5)

### 2026-07-16 - Missing userId = failed eKYC check
- **Changes**: Prod logs showed workflows completing with empty extraction (`userId(documentNumber)=, fullname=`) → MTSS rejected with `-103 Missing userId` → RuntimeException + misleading ERROR stack trace, even though the real cause is a failed eKYC (Jumio couldn't extract the document). Fix: `MtssServiceImpl.callCallbackJumio` now checks `documentNumber` right after extraction — if blank, logs `eKYC FAILED — userId (documentNumber) not found...` at WARN, skips the MTSS SOAP call, returns `false` (signature changed `void` → `boolean` in `MtssService` interface). `JumioCallbackController.handleProcessed` logs the outcome accordingly; original payload still forwarded to Adacash unchanged. Also committed the previously uncommitted JumioClient 401 fix as its own commit. Verified with `mvn compile`, pushed both commits (`1050e82` 401 fix, `03bf023` userId check) to `origin/feature/per-session-callback-url`. **Redeploy to PROD still pending.**
- **Time Spent**: ~30 min

### 2026-07-09 - OAuth 401 token bug fix (Jumio-reported)
- **Changes**: Jumio reported intermittent HTTP 401 on `JumioAccountFeignClient#initiateWorkflow` for several `userReference`/borrowerIds (632780, 1241776, 1241760, etc.), causing empty `webHref` on session creation. Diagnosed from proxy logs (`issue no 4.txt`) — root cause: `JumioClient.java`'s per-project token cache had no synchronized refresh (race condition on concurrent cache miss) and no 401 retry (a rejected/expired Bearer token failed the call outright instead of recovering). Jumio sent a suggested fix, but their sample was single-tenant (global `JumioProperties`, single `cachedToken` field) — confirmed via git archaeology that `master` on the `jumio-proxy` repo (a separate nested repo at `trustgate/jumio-proxy.git`) is single-tenant, while production actually runs `feature/per-session-callback-url` (multi-tenant, per-project token cache keyed by `projectId`). Adapted the fix to preserve multi-tenancy: added per-project `synchronized` lock on cache-miss token fetch, and wrapped `initiateWorkflow`/`getWorkflowExecutionRaw` in a `executeWithAuthRetry` helper that catches `JumioFeignException` on status 401, invalidates that project's cached token, and retries once. Built and deployed to production. Replied to Jumio explaining the multi-tenant adaptation.
- **Time Spent**: ~1.5 hours

### 2026-06-26 - PROD Deployed & Tested ✅
- **Changes**: Docker image built and pushed to registry (`localhost:30445/jumioproxy:1.0`). Namespace `jumioproxy` created. `configmap.yaml` + `deployment.yaml` applied to K8s. Nginx updated with `jumioproxyProd` upstream (port 30240) + location block, reloaded. Adacash re-tested on PROD URL — confirmed working end-to-end.
- **Time Spent**: ~30 min

### 2026-06-23 - PROD deployment YAML prep
- **Changes**: Prepared full production deployment package — rewrote `production/deployment.yaml` (namespace `jumioproxy`, image `jumioproxy:1.0`, `imagePullPolicy: Always`, NodePort 30240, prod API keys). Updated `production/configmap.yaml` (prod namespace, prod callback-base-url, prod MTSS endpoint `MyTrustSignerService`, MTSS creds: `RockWingEL_prod`/`RockWing`, Adacash client-callback-url `https://jumio.adacash.my/api/v2/postback`). Created `production/nginx/nginx-prod.conf` (location block + upstream `jumioproxyProd`). Clarified API key design — global list, not per-project; new tenant = new key appended to comma-separated env var. Confirmed via `ApiKeyFilter.java`.
- **Time Spent**: ~45 min

### 2026-06-19 - callbackUrl debug + imagePullPolicy fix
- **Changes**: Diagnosed v1 pilot not forwarding to per-session `callbackUrl` — root cause was `imagePullPolicy: IfNotPresent` causing K8s to use cached old image (built from `feature/multitenant-support`, no SessionCallbackStore) instead of new image with callbackUrl feature. Fix: set `imagePullPolicy: Always`, cycled tag 1.07 → 1.08 to force fresh pull. Both v1 (NodePort 30241) and v2 (NodePort 30242) confirmed working end-to-end — MTSS SOAP success, forward to Adacash `callbackUrl` success.
- **Time Spent**: ~45 min

## Historical Summary
Project started 2026-04-16 as Trustgate proxy layer between Adacash and Jumio eKYC API. Key milestones: full callback flow implemented (MTSS SOAP, image download, Adacash forward), image compression added, SESSION_EXPIRED bug fixed. Reactivated 2026-06-04 for multi-tenant support — path-based routing designed and coded. Master restored to clean single-tenant. pilot-v2 deployed to K8s (namespace `jumioproxy-pilot`, NodePort 30242). Per-session callbackUrl feature implemented and tested. pilot v1 migrated to use v2 multi-tenant JAR. PROD deployed and Adacash-tested 2026-06-26. Per-session callbackUrl feature implemented 2026-06-18 (`SessionCallbackStore`, optional `callbackUrl` in `CreateSessionRequest`, `resolveCallbackUrl()` fallback to ConfigMap) and pilot v1 migrated to the v2 multi-tenant JAR the same day. Reactivated again 2026-07-09 after Jumio reported intermittent 401 auth errors — fixed with per-project synchronized token refresh + 401 retry-once logic, preserving multi-tenant architecture.

## Technical Notes
- **Repository**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\app\jumio-proxy\` — this is a separate nested git repo (`trustgate/jumio-proxy.git`), distinct from the outer `docker` repo (`trustgate/docker.git`)
- **Artifact ID**: `adacash-trustgate-jumio-proxy` v0.0.1
- **Git branches**: `master` (single-tenant, stale), `feature/multitenant-support` (multi-tenant), `feature/per-session-callback-url` (callbackUrl feature — based on multitenant, **this is what production actually runs**)
- **pilot folder**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\pilot\` — NodePort 30241, image `jumioproxy-pilot:1.08` (`imagePullPolicy: Always`), context `/jumio-proxy`
- **pilot-v2 folder**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\pilot-v2\` — NodePort 30242, image `jumioproxy-pilot-v2:1.02`, context `/jumio-proxy`
- **Prod folder**: `production/` — `deployment.yaml` + `configmap.yaml` + `nginx/nginx-prod.conf` — all ready ✅
- **Prod image**: `localhost:30445/jumioproxy:1.0`
- **Prod namespace**: `jumioproxy`
- **Nginx**: Combined location block handles both `/api/v1/` and `/actuator/` via dual rewrite directives
- **ConfigMap**: single source of truth for per-project Jumio + MTSS + clientCallbackUrl config
- **API key design**: Global list (`JUMIO_PROXY_API_KEYS`), not per-project. New tenant = append new key. Validated by `ApiKeyFilter.java` against `X-Api-Key` header.
- **Callback URL (pilot)**: `https://digitalid.msctrustgate.com/jumioproxy_pilot/adacash/api/v1/jumio/callback`
- **Callback URL (pilot-v2)**: `https://digitalid.msctrustgate.com/jumioproxy_pilotv2/adacash/api/v1/jumio/callback`
- **Callback URL (prod)**: `https://digitalid.msctrustgate.com/jumioproxy/adacash/api/v1/jumio/callback`
- **Session endpoint (prod)**: `POST https://digitalid.msctrustgate.com/jumioproxy/adacash/api/v1/jumio/session`
- **Retrieval endpoint (prod)**: `GET https://digitalid.msctrustgate.com/jumioproxy/adacash/api/v1/jumio/accounts/{accountId}/workflow-executions/{workflowId}` — returns raw Jumio JSON with `credentials[].parts[].href` links; no image-download REST endpoint on the proxy itself, `href`s must be called directly against Jumio's retrieval servers with a Bearer token
- **JumioClient.java 401 fix (2026-07-09)**: per-project `synchronized` lock (`tokenLocks` map) on cache-miss token fetch; `executeWithAuthRetry()` wraps `initiateWorkflow`/`getWorkflowExecutionRaw` — on `JumioFeignException` status 401, invalidates that project's cached token and retries once

---
**Last Updated**: 2026-07-16 | **Status**: Active — #1 — userId-check pushed, PROD redeploy pending; monitoring for 401 recurrence
