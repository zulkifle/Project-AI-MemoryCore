---
name: tgekyc-liveness-deployment-checklist
description: "MUST use when working on the tgekyc liveness service deployment —
             redeploying, upgrading the vendor image version, checking its K8s
             config, verifying the deployed version via /version.json, or
             diagnosing 'OpenAI API key not configured' /
             FAKE_FACE errors from MPAY's doLivenessVideo flow. Also triggers on:
             'tgekyc liveness', 'liveness deployment', 'check liveness config',
             'redeploy liveness', 'upgrade liveness image', 'tgekyc-live',
             'check liveness version', 'confirm deployed version'."
---

# TG eKYC Liveness Deployment Checklist
*Known config + verification steps for the Ctrl CV liveness service running in the tgekyc K8s environment.*

## Activation

When this skill activates, output:

`✅ TG eKYC Liveness Deployment Checklist — loading known config...`

Then execute the protocol below.

## Context Guard

| Context | Status |
|---------|--------|
| **User is redeploying/upgrading the tgekyc liveness image** | ACTIVE — full checklist |
| **User is debugging a liveness/authenticity error from MPAY** | ACTIVE — troubleshooting section |
| **User asks to check the tgekyc liveness config** | ACTIVE — report known config below |
| **A different eKYC vendor or a non-tgekyc K8s deployment** | DORMANT — this is tgekyc-specific, not a general K8s skill |

---

## Known Configuration

| Item | Value |
|---|---|
| Namespace | `tgekyc` (moved from `tgekyc-staging` 2026-08-03 — that name was legacy, this service has been running in production; shares the namespace with the OCR `tgekyc` deployment. Per-pod resource requests/limits don't pool across a namespace, so this doesn't constrain the OCR pod — confirmed no `ResourceQuota`/`LimitRange` exists on either namespace) |
| Deployment name | `tgekyc-live` (renamed from `tgekyc-live-stag` 2026-08-03 — dropped "-stag", this is production) |
| Service / NodePort | `tgekyc-live`, port `5010`, nodePort `30031` (unchanged — external callers use the NodePort number, not the Service name) |
| Image (as of 2026-08-01) | `localhost:30445/tgekyc-liveness:1.063` |
| CPU (request / limit) | `2` / `4` cores — matches vendor's v1.3.1 spec (peaks ~3 cores during decode, settles ~2 while running; vendor recommends 2-4 per pod assuming 1 request at a time) |
| Memory (request / limit) | `3Gi` / `6Gi` (lowered from `10Gi`/`20Gi` 2026-08-03 per vendor spec — v1.3.1 uses ~1.8GiB at 5-6s/30fps, ~3.2GiB at 7s/60fps, ~4.9GiB hard max at ~600 frames) |
| Manifest path | `C:\PROJECTS\DOCKER GITLAB\docker\tgekyc\deployment-live.yaml` (renamed from `deployment-live-stag.yaml` 2026-08-03) |
| Vendor build source | `C:\PROJECTS\EKYC\Deployment\liveness_detection-master` |
| Vendor setup doc | `C:\Users\opera\OneDrive\Desktop\KUBERNETES.md` (Ctrl CV) |
| Secret (OpenAI key) | `liveness-openai`, key `OPENAI_API_KEY`, must exist in the **`tgekyc`** namespace (moved with the deployment 2026-08-03 — namespace-scoped, does not follow automatically, must be recreated/copied there) |
| `SAVE_DATA` | `"0"` — deliberately off, no biometric video retention (Dejul's call, 2026-08-01) |
| Probes | **Not configured** — Dejul explicitly declined `/version.json` readiness/liveness probes (2026-08-01). Do not silently add them back; ask first if revisiting. |
| Calling system | MPAY `doLivenessVideo` (Third Party Integration API) |

---

## Protocol — Redeploy / Upgrade Image

1. Confirm the target manifest: `deployment-live.yaml` for the liveness service (other files in the folder — `deployment.yaml`, `deployment-dev.yaml`, `deployment-staging.yaml` — are older OCR-only tgekyc services, not the liveness image; don't confuse them).
2. Bump the `image:` tag to the new version.
3. **Before applying**, re-check the vendor's `KUBERNETES.md` (or newer version if the client sent an update) for any *new* required env vars — vendor packages like this add requirements between versions.
4. Confirm `OPENAI_API_KEY` is still wired via `secretKeyRef` (not a bare value, not `.env` copied in — Kubernetes does not read `.env` files, that's docker-compose-only behavior), and that the `liveness-openai` Secret exists in the **`tgekyc`** namespace.
5. Confirm `SAVE_DATA` is still `"0"` unless Dejul has explicitly asked to retain videos.
6. `kubectl apply -f deployment-live.yaml` — picks up the pod-spec change and rolls automatically.
7. Verify per the steps below before calling it done.

### One-time migration note (2026-08-03: `tgekyc-staging`/`tgekyc-live-stag` → `tgekyc`/`tgekyc-live`)

Renaming a Deployment/Service means new objects, not an in-place rename — and the old Service was holding NodePort `30031`, which the new Service also needs, so the old one must be freed first:

```bash
# 1. Copy the Secret into the new namespace (namespace-scoped, doesn't move on its own)
kubectl get secret liveness-openai -n tgekyc-staging -o yaml | sed 's/namespace: tgekyc-staging/namespace: tgekyc/' | kubectl apply -f -

# 2. Delete the OLD Service first — frees nodePort 30031 for the new one
kubectl delete service tgekyc-live-stag -n tgekyc-staging

# 3. Apply the new manifest (creates Deployment + Service in the tgekyc namespace)
kubectl apply -f deployment-live.yaml

# 4. Clean up the old Deployment
kubectl delete deployment tgekyc-live-stag -n tgekyc-staging

# 5. Verify — see Verify/Troubleshoot and Verify Deployed Version below
```

Expect a brief gap in service between steps 2 and 3 (single replica, no rolling handover across a rename) — same downtime profile as any other redeploy of this single-replica service.

## Protocol — Verify / Troubleshoot

```bash
# 1. Key reached the container? (prints length, never the key itself)
kubectl exec -n tgekyc deploy/tgekyc-live -- sh -c 'echo ${#OPENAI_API_KEY}'
# 0 = did NOT reach the container — check Secret exists in tgekyc namespace and name/key match the manifest exactly

# 2. Pod healthy
kubectl get pods -n tgekyc -l app=tgekyc-live
kubectl logs -n tgekyc deploy/tgekyc-live --tail=100

# 3. Real end-to-end test — run an actual liveness video through MPAY's doLivenessVideo flow
```

Read `reasoning` / `confidence_reason` in the response:

| Symptom | Cause |
|---|---|
| `"OpenAI API key not configured"` | Secret missing, wrong name/key, or not in `tgekyc` namespace, or pod not restarted after Secret was created |
| `service_error: api_error` | Key present but the OpenAI call failed — invalid/expired key, no egress from the pod, proxy/NetworkPolicy blocking, rate limit |
| `service_error: no_frames` | Uploaded video couldn't be decoded — corrupt/empty/unsupported format |
| Pod `OOMKilled` | Memory limit too low for the video length/resolution, or too many concurrent requests on one pod — limits (`3Gi`/`6Gi`, set 2026-08-03) now track the vendor's own v1.3.1 spec closely (hard max ~4.9GiB at ~600 frames), so this is more plausible than it was under the old 20Gi ceiling — check frame count/duration of the failing video first |

## Protocol — Verify Deployed Version

Vendor-documented endpoint, confirmed working. Full guide: `C:\PROJECTS\EKYC\Deployment\how-to-call-version.md`.

`GET /version` (readable page) / `GET /version.json` (JSON) — plain GET, no auth, no request body, never cached. Container port is `5010` (matches the service port above).

```bash
# From inside the cluster
curl http://tgekyc-live.tgekyc.svc.cluster.local/version.json

# Or via port-forward
kubectl port-forward -n tgekyc deploy/tgekyc-live 5010:5010
curl http://localhost:5010/version.json
# then open http://localhost:5010/version in a browser for the readable page

# Or via ingress, if exposed — same host used for liveness requests
curl https://<your-host>/version
```

Expected response:

```json
{
  "data": {
    "product": "Ctrl CV Liveness",
    "system_version": "v1.3.1",
    "openai_model": "gpt-4o-2024-08-06",
    "prompt_package": "v1.2.1",
    "prompt_sha256": "e236a4641f9ae5aafa57c270dbe5411c4394d1ac2449f9d526400e36a4b74839",
    "frame_processor": "v1.1.2",
    "decision_engine": "v1.0.1",
    "backend_commit": "4e6688d",
    "server_time": "2026-08-01 15:53:14 +0800"
  },
  "code": 200
}
```

| Field | Meaning |
|---|---|
| `system_version` | **the one to check** — release identifier, should match the release notes. `"unknown"` means the running build predates v1.3.1 |
| `openai_model` | exact dated model snapshot in use, not a floating alias |
| `prompt_package` / `prompt_sha256` | detection instruction set version / fingerprint (sha changes if the prompt changes) |
| `frame_processor` | video decoding / frame-prep component version |
| `decision_engine` | pass/decline logic version |
| `backend_commit` | source revision the release was built from |
| `server_time` | server time, Malaysia local |

Notes:
- Response is never cached — always reflects the build actually running. A browser tab open from before an upgrade needs a hard refresh (Ctrl+Shift+R), not a trust-the-cache assumption.
- Safe as a K8s readiness/liveness probe target (responds in every configuration) — but see the Known Configuration table: probes are deliberately **not** configured here; don't add this as a probe without asking first.
- `GET /ping` is a lighter health check (success/fail only) — use `/version.json` when you need to confirm *which* build is answering, not just that it's up.

**Use this after every redeploy/upgrade** (step 7 of the protocol above) to confirm `system_version` actually changed to the intended tag, instead of assuming `kubectl apply` succeeded from exit code alone.

---

## Mandatory Rules

1. **This is tgekyc-specific** — do not generalize this skill's steps to other vendors' K8s deployments; if a different eKYC/liveness vendor comes up, treat it as a new investigation, not this checklist
2. **Never assume `.env` reaches a K8s pod** — always check for a Secret + `secretKeyRef`, regardless of what the build folder's `.env` contains
3. **`SAVE_DATA` stays `"0"` unless Dejul explicitly says otherwise** — this is biometric data retention, a compliance-sensitive decision, not a default to flip casually
4. **Don't silently add probes back** — Dejul declined them once; ask before reintroducing, don't treat their absence as a bug to auto-fix
5. **Re-read the vendor's current `KUBERNETES.md` before every image upgrade** — assume requirements can change between versions, don't rely on this file's snapshot alone
6. **Namespace is `tgekyc`, not `tgekyc-staging`** — this service is production, the old name was leftover from before it went live; if old references to `tgekyc-staging`/`tgekyc-live-stag` turn up elsewhere (scripts, docs, other manifests), they're stale, not a sign something's misconfigured
7. **Resource limits (`3Gi`/`6Gi` mem, `2`/`4` CPU) came from the vendor's own v1.3.1 spec, not a guess** — don't loosen them back toward the old `10Gi`/`20Gi` without a reason; if OOMKilled turns up, check the vendor's per-frame numbers before just raising the ceiling

---

## Edge Cases

| Situation | Behavior |
|---|---|
| `deployment.yaml` / `deployment-dev.yaml` / `deployment-staging.yaml` mentioned | These are the older tgekyc OCR service manifests, unrelated to the liveness image — confirm which manifest before editing anything |
| User asks to enable video retention | Confirm explicitly this is a deliberate compliance decision before flipping `SAVE_DATA` to `"1"` |
| Vendor sends a new `.env` file for an upgrade | Same trap as before — extract the value, put it in the `liveness-openai` Secret, never mount/copy `.env` into the pod |
| `echo ${#OPENAI_API_KEY}` returns non-zero but errors still occur | Key reached the container but may be invalid/expired/wrong project — check `service_error` field, not just presence |
| Old commands/docs reference `tgekyc-staging` namespace or `tgekyc-live-stag` name | Stale — migrated 2026-08-03 to `tgekyc`/`tgekyc-live`. Update the reference, don't assume a second environment exists |

---

## Level History

- **Lv.1** — Base: known config table, redeploy/upgrade protocol, verify/troubleshoot protocol, mandatory rules. (Origin: 2026-08-01, first deployment fix — `OPENAI_API_KEY` not reaching the pod because K8s doesn't read `.env`, `SAVE_DATA` disabled per Dejul, probes declined)
- **Lv.2** — Verify Deployed Version protocol: vendor-documented `/version` and `/version.json` endpoint (port 5010, no auth, never cached), expected response fields, in-cluster/port-forward/ingress call forms, `system_version` as the field to check. (Origin: 2026-08-03, vendor guide `C:\PROJECTS\EKYC\Deployment\how-to-call-version.md`)
- **Lv.3** — Production migration: renamed `tgekyc-staging`/`tgekyc-live-stag` → `tgekyc`/`tgekyc-live` (service was already running in production, old name was leftover); moved into the existing `tgekyc` namespace alongside the OCR deployment after confirming per-pod resource limits don't pool across a namespace; lowered memory to `3Gi`/`6Gi` (from `10Gi`/`20Gi`) per the vendor's v1.3.1 frame-count spec, CPU confirmed unchanged at `2`/`4`. Added the delete-old-Service-first migration sequence (NodePort reuse) as a one-time note. (Origin: 2026-08-03, client email with per-request memory/CPU breakdown)
