# jumio-proxy-integration - Adacash–Trustgate Jumio Proxy Service
*Spring Boot proxy layer between Adacash and Jumio for eKYC integration*

## Project Overview
- **Type**: Web Service / Proxy API (Spring Boot)
- **Client**: Adacash Sdn Bhd
- **Period**: 2026-04-16 - Active
- **Tech Stack**: Java 17 + Spring Boot 3.0.9 + OpenFeign + Maven + Docker + Kubernetes
- **Completion**: 100% (maintenance)
- **Duration**: ~20.5 hours
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
- **Last Session**: 2026-08-19 - Alpha-channel fix deployed to real production namespace `jumioproxy`; repo consolidated to a single `master` branch
- **Next Steps**: None open — fix live in production, repo cleaned up, branch confusion resolved
- **Known Issues**: None open — fix verified (113/113 bulk recovery) and now live in real production

## Session History (Last 5)

### 2026-08-19 - Deployed to real production + consolidated repo to a single master branch
- **Changes**: Deployed the `fix/cmyk-jpeg-compression` build to the real `jumioproxy` production namespace (previously only verified in the `jumioproxy-test` pre-prod lane) — new incoming Jumio callbacks through true prod are now covered by the alpha-channel fix. Separately, consolidated the repo: confirmed `fix/cmyk-jpeg-compression` fully contained both `feature/multitenant-support` and `feature/per-session-callback-url` as ancestors, so merged everything into `master` and deleted both feature branches (local + remote) — Zul is sole developer on this repo, no collaboration risk. First attempt to reset `master` and force-push was rejected by GitLab's branch protection (force-push disabled on `master`, correctly). A `git pull` in between then produced real merge conflicts against master's old diverged history (which had its own separate, abandoned "restore single-tenant" commit). Resolved properly via `git merge origin/master -X ours` — a normal, non-destructive merge commit favoring our side wherever content actually conflicted. One genuine leftover from the stale history slipped through as a clean auto-merge: a duplicated `image: compression:` block in `application.yml` — removed and amended into the merge commit. Verified `mvn compile` clean, pushed to `origin/master`, deleted the two stale remote branches. Repo is now just `master`, containing the multi-tenant architecture, per-session callbackUrl, 401 retry fix, userId/failed-eKYC check, and the alpha-channel compression fix, all in one place.
- **Time Spent**: ~45 min

### 2026-08-18 - Root-caused JPEG alpha-channel compression bug + built manual recovery API + bulk-fixed 113 stuck records
- **Changes**: Chased a `"Bogus input colorspace"` compression failure in `ImageCompressionUtil` that was silently returning oversized original images, causing MTSS SOAP calls to fail with `413 Request Entity Too Large`. Initial theory (CMYK JPEG) led to adding TwelveMonkeys `imageio-jpeg:3.10.1` + explicit reader-selection code (`readImagePreferringTwelveMonkeys()`) — confirmed via local repro that TwelveMonkeys registers and wins correctly, but the error persisted. True root cause found by pulling the actual raw image bytes via Postman and inspecting with ImageMagick/exiftool: Jumio's FRONT image for this record is genuinely a **PNG with an alpha channel (RGBA)**, not JPEG at all — `compress()` always re-encodes to JPEG regardless of input format, and JPEG has no alpha channel; writing an ARGB `BufferedImage` through a JPEG `ImageWriter` throws that exact error. Reproduced standalone with a synthetic ARGB image to confirm bug + fix in isolation. Fix: new `toRgb()` flattens alpha onto a white background before the JPEG write. Also added a new manual recovery endpoint `POST /api/v1/{projectId}/jumio/accounts/{accountId}/workflow-executions/{workflowId}/reprocess-images` (re-downloads/recompresses/re-pushes to MTSS for a specific stuck record; shared retrieval logic extracted out of `handleProcessed()`). Diagnosed and worked around several K8s deploy-freshness traps along the way — reusing the same image tag across rebuilds means `kubectl apply` sees no spec diff and never restarts the pod even with `imagePullPolicy: Always`, needing `rollout restart` or a genuinely new tag each time. Confirmed the Nginx per-tenant rewrite convention (`/jumioproxy[-test]/{tenant}/api/v1/... → /jumio-proxy/api/v1/{tenant}/...`) via the real config Zul added for a new `production-test/` K8s + Nginx lane (namespace `jumioproxy-test`, mirrors real prod `adacash` Jumio/MTSS credentials — used as a pre-prod test lane). Verified the fix there, then wrote a PowerShell batch script parsing Zul's `jumio-failed-user.txt` backlog (123 lines, 113 unique accountId/workflowId records spanning 11–18 Aug) and looped the new reprocess-images API against real production — **all 113 succeeded, 0 failures**.
- **Time Spent**: ~3.5 hours

### 2026-07-16 - Missing userId = failed eKYC check
- **Changes**: Prod logs showed workflows completing with empty extraction (`userId(documentNumber)=, fullname=`) → MTSS rejected with `-103 Missing userId` → RuntimeException + misleading ERROR stack trace, even though the real cause is a failed eKYC (Jumio couldn't extract the document). Fix: `MtssServiceImpl.callCallbackJumio` now checks `documentNumber` right after extraction — if blank, logs `eKYC FAILED — userId (documentNumber) not found...` at WARN, skips the MTSS SOAP call, returns `false` (signature changed `void` → `boolean` in `MtssService` interface). `JumioCallbackController.handleProcessed` logs the outcome accordingly; original payload still forwarded to Adacash unchanged. Also committed the previously uncommitted JumioClient 401 fix as its own commit. Verified with `mvn compile`, pushed both commits (`1050e82` 401 fix, `03bf023` userId check) to `origin/feature/per-session-callback-url`. Same night (~midnight): Docker image rebuilt and **redeployed to both PILOT and PROD** ✅. Log monitoring done and Jumio confirmation received — 401 issue considered resolved. Only branch merge remains.
- **Time Spent**: ~45 min

### 2026-07-09 - OAuth 401 token bug fix (Jumio-reported)
- **Changes**: Jumio reported intermittent HTTP 401 on `JumioAccountFeignClient#initiateWorkflow` for several `userReference`/borrowerIds (632780, 1241776, 1241760, etc.), causing empty `webHref` on session creation. Diagnosed from proxy logs (`issue no 4.txt`) — root cause: `JumioClient.java`'s per-project token cache had no synchronized refresh (race condition on concurrent cache miss) and no 401 retry (a rejected/expired Bearer token failed the call outright instead of recovering). Jumio sent a suggested fix, but their sample was single-tenant (global `JumioProperties`, single `cachedToken` field) — confirmed via git archaeology that `master` on the `jumio-proxy` repo (a separate nested repo at `trustgate/jumio-proxy.git`) is single-tenant, while production actually runs `feature/per-session-callback-url` (multi-tenant, per-project token cache keyed by `projectId`). Adapted the fix to preserve multi-tenancy: added per-project `synchronized` lock on cache-miss token fetch, and wrapped `initiateWorkflow`/`getWorkflowExecutionRaw` in a `executeWithAuthRetry` helper that catches `JumioFeignException` on status 401, invalidates that project's cached token, and retries once. Built and deployed to production. Replied to Jumio explaining the multi-tenant adaptation.
- **Time Spent**: ~1.5 hours

### 2026-06-26 - PROD Deployed & Tested ✅
- **Changes**: Docker image built and pushed to registry (`localhost:30445/jumioproxy:1.0`). Namespace `jumioproxy` created. `configmap.yaml` + `deployment.yaml` applied to K8s. Nginx updated with `jumioproxyProd` upstream (port 30240) + location block, reloaded. Adacash re-tested on PROD URL — confirmed working end-to-end.
- **Time Spent**: ~30 min

## Historical Summary
Project started 2026-04-16 as Trustgate proxy layer between Adacash and Jumio eKYC API. Key milestones: full callback flow implemented (MTSS SOAP, image download, Adacash forward), image compression added, SESSION_EXPIRED bug fixed. Reactivated 2026-06-04 for multi-tenant support — path-based routing designed and coded. Master restored to clean single-tenant. pilot-v2 deployed to K8s (namespace `jumioproxy-pilot`, NodePort 30242). Per-session callbackUrl feature implemented and tested. pilot v1 migrated to use v2 multi-tenant JAR. Full production deployment package prepared 2026-06-23 (`production/deployment.yaml` + `configmap.yaml` + `nginx-prod.conf`, prod MTSS creds `RockWingEL_prod`/`RockWing`); PROD deployed and Adacash-tested 2026-06-26. Per-session callbackUrl feature implemented 2026-06-18 (`SessionCallbackStore`, optional `callbackUrl` in `CreateSessionRequest`, `resolveCallbackUrl()` fallback to ConfigMap) and pilot v1 migrated to the v2 multi-tenant JAR the same day — that same day, diagnosed pilot v1 not forwarding to per-session `callbackUrl` at all, caused by `imagePullPolicy: IfNotPresent` serving a stale cached image; fixed by switching to `imagePullPolicy: Always` and cycling the tag to force a fresh pull. Reactivated again 2026-07-09 after Jumio reported intermittent 401 auth errors — fixed with per-project synchronized token refresh + 401 retry-once logic, preserving multi-tenant architecture.

## Technical Notes
- **Repository**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\app\jumio-proxy\` — this is a separate nested git repo (`trustgate/jumio-proxy.git`), distinct from the outer `docker` repo (`trustgate/docker.git`)
- **Artifact ID**: `adacash-trustgate-jumio-proxy` v0.0.1
- **Git branches (as of 2026-08-19)**: consolidated to just `master` — `feature/multitenant-support` and `feature/per-session-callback-url` deleted (local + remote) after confirming both were fully-contained ancestors of the final fix branch. `master` now contains the multi-tenant architecture, per-session callbackUrl, 401 retry fix, userId/failed-eKYC check, and the alpha-channel compression fix. Note: `master` had branch protection against force-push on GitLab — history was reconciled via a real merge commit (`-X ours`), not a rewrite
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
- **Manual recovery endpoint (2026-08-18)**: `POST /api/v1/{projectId}/jumio/accounts/{accountId}/workflow-executions/{workflowId}/reprocess-images?userReference=` in `JumioCallbackController` — re-downloads FRONT/BACK/liveness(FACE) images, recompresses, re-pushes to MTSS on demand for a specific stuck record; returns `{success, message, frontImageBytes, backImageBytes, selfieImageBytes}`
- **ImageCompressionUtil fix (2026-08-18)**: `toRgb()` flattens alpha channel onto white before the JPEG write — the actual fix for `"Bogus input colorspace"`, since Jumio can serve FRONT/BACK/FACE as PNG-with-alpha despite the proxy always re-encoding to JPEG. `readImagePreferringTwelveMonkeys()` (explicit TwelveMonkeys `imageio-jpeg:3.10.1` reader selection) was added first chasing a CMYK-JPEG theory that turned out wrong for this specific bug, but is kept as a defensive improvement for genuine CMYK JPEG inputs
- **K8s namespaces**: `jumioproxy` (real production) and `jumioproxy-test` (`production-test/` folder — new as of 2026-08-18, mirrors real prod `adacash` Jumio/MTSS credentials, used as a pre-prod test lane before touching real `jumioproxy`)
- **K8s deploy gotcha**: reusing the same image tag across rebuilds means `kubectl apply` sees no Deployment spec diff and never restarts the pod, even with `imagePullPolicy: Always` — always bump the tag or run `kubectl rollout restart deployment/jumioproxy -n <namespace>` after pushing
- **Nginx per-tenant rewrite convention**: `^/jumioproxy[-test]/([^/]+)/api/v1/(.*)$` → `/jumio-proxy/api/v1/$1/$2` — external URLs put the tenant once before `/api/v1/`, Nginx reinserts it into the `{projectId}` path slot

---
**Last Updated**: 2026-08-19 | **Status**: Active — #1 — Alpha-channel JPEG compression fix live in real `jumioproxy` production namespace; repo consolidated to a single `master` branch; nothing outstanding
