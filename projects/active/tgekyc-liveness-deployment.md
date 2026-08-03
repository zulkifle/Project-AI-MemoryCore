# TG eKYC Liveness Deployment - Vendor liveness package deployed to internal K8s
*Deploying Ctrl CV's liveness detection service (biometric authenticity check) into the tgekyc K8s environment, wired into the existing MPAY doLivenessVideo integration.*

## Project Overview
- **Type**: Integration / Deployment
- **Client**: Internal (Trustgate) — vendor package from Ctrl CV
- **Period**: 2026-08-01 - Active
- **Tech Stack**: Backend: Python 3.12 Flask (vendor image, `tgekyc-liveness`) + **Frontend**: N/A + **Database**: N/A (stateless, K8s Deployment/Service)
- **Completion**: 60%
- **Duration**: ~40 min
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-08-01 - Fixed `OPENAI_API_KEY` not reaching the pod; disabled video retention
- **Next Steps**:
  1. Confirm with `kubectl exec ... echo ${#OPENAI_API_KEY}` that the key reached the container
  2. Run a real liveness video through MPAY's `doLivenessVideo` flow and confirm `reasoning`/`confidence_reason` no longer says "OpenAI API key not configured"
  3. Probes (`/version.json` readiness/liveness) — explicitly declined by user for now, revisit if pod restart-loop issues come up
- **Known Issues**: None open — key + `SAVE_DATA` fixes applied and deployed by Dejul

## Session History (Last 5)

### 2026-08-01 - Diagnosed and fixed missing OPENAI_API_KEY in staging deployment
- **Changes**: MPAY's `doLivenessVideo` call was returning `"reasoning":"OpenAI API key not configured"` / `is_live:false` for every authenticity check. Traced to `C:\PROJECTS\DOCKER GITLAB\docker\tgekyc\deployment-live-stag.yaml` (namespace `tgekyc-staging`, image `tgekyc-liveness:1.062`) — the vendor's own `KUBERNETES.md` (`C:\Users\opera\OneDrive\Desktop\KUBERNETES.md`) explicitly warns Kubernetes does not read `.env` files, so the real key sitting in the build folder's `.env` (`C:\PROJECTS\EKYC\Deployment\liveness_detection-master\.env`) never reached the pod. Edited the manifest: added `OPENAI_API_KEY` via `secretKeyRef` (`liveness-openai` Secret, `tgekyc-staging` namespace), flipped `SAVE_DATA` from `"1"` to `"0"` per Dejul's confirmation that biometric video retention isn't wanted yet. Readiness/liveness probes on `/version.json` (also flagged by the vendor doc) were proposed but Dejul declined adding them for now — left out of the manifest. Dejul created the K8s Secret and ran `kubectl apply` himself; deployment confirmed done.
- **Time Spent**: ~40 min

## Historical Summary
No history yet — this section is populated when session count exceeds 5.

## Technical Notes
- **Repository**: Vendor build source: `C:\PROJECTS\EKYC\Deployment\liveness_detection-master` | K8s manifests: `C:\PROJECTS\DOCKER GITLAB\docker\tgekyc\` (`deployment.yaml`, `deployment-dev.yaml`, `deployment-staging.yaml`, `deployment-live-stag.yaml` — staging liveness lives in the last one)
- **Namespace**: `tgekyc-staging` | **Deployment**: `tgekyc-live-stag` | **Image**: `localhost:30445/tgekyc-liveness:1.062` | **Port**: 5010 (NodePort 30031)
- **Secret**: `liveness-openai` (key `OPENAI_API_KEY`) — must exist in `tgekyc-staging` namespace before `kubectl apply`, created manually by Dejul (not committed to any repo, per vendor's handling guidance)
- **Key Dependencies**: OpenAI vision model API (authenticity/verification_type 4 only — other 4 endpoints unaffected by the key), MPAY third-party integration API (calls `doLivenessVideo`)
- **Vendor doc reference**: `C:\Users\opera\OneDrive\Desktop\KUBERNETES.md` — authoritative setup/troubleshooting guide from Ctrl CV; also documents `/version.json` vs `/ping` probe gotcha and memory sizing (6Gi limit, `max_frame_length` tuning)

---
**Last Updated**: 2026-08-01 | **Position**: #1/10 Active
