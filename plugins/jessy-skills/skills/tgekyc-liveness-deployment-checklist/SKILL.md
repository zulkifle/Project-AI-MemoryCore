---
name: tgekyc-liveness-deployment-checklist
description: "MUST use when working on the tgekyc liveness service deployment —
             redeploying, upgrading the vendor image version, checking its K8s
             config, or diagnosing 'OpenAI API key not configured' /
             FAKE_FACE errors from MPAY's doLivenessVideo flow. Also triggers on:
             'tgekyc liveness', 'liveness deployment', 'check liveness config',
             'redeploy liveness', 'upgrade liveness image', 'tgekyc-live-stag'."
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
| Namespace | `tgekyc-staging` |
| Deployment name | `tgekyc-live-stag` |
| Service / NodePort | `tgekyc-live-stag`, port `5010`, nodePort `30031` |
| Image (as of 2026-08-01) | `localhost:30445/tgekyc-liveness:1.063` |
| Memory (request / limit) | `10Gi` / `20Gi` |
| Manifest path | `C:\PROJECTS\DOCKER GITLAB\docker\tgekyc\deployment-live-stag.yaml` |
| Vendor build source | `C:\PROJECTS\EKYC\Deployment\liveness_detection-master` |
| Vendor setup doc | `C:\Users\opera\OneDrive\Desktop\KUBERNETES.md` (Ctrl CV) |
| Secret (OpenAI key) | `liveness-openai`, key `OPENAI_API_KEY`, must exist in `tgekyc-staging` |
| `SAVE_DATA` | `"0"` — deliberately off, no biometric video retention (Dejul's call, 2026-08-01) |
| Probes | **Not configured** — Dejul explicitly declined `/version.json` readiness/liveness probes (2026-08-01). Do not silently add them back; ask first if revisiting. |
| Calling system | MPAY `doLivenessVideo` (Third Party Integration API) |

---

## Protocol — Redeploy / Upgrade Image

1. Confirm the target manifest: `deployment-live-stag.yaml` for staging (other envs — `deployment.yaml`, `deployment-dev.yaml`, `deployment-staging.yaml` — are older OCR-only tgekyc services, not the liveness image; don't confuse them).
2. Bump the `image:` tag to the new version.
3. **Before applying**, re-check the vendor's `KUBERNETES.md` (or newer version if the client sent an update) for any *new* required env vars — vendor packages like this add requirements between versions.
4. Confirm `OPENAI_API_KEY` is still wired via `secretKeyRef` (not a bare value, not `.env` copied in — Kubernetes does not read `.env` files, that's docker-compose-only behavior).
5. Confirm `SAVE_DATA` is still `"0"` unless Dejul has explicitly asked to retain videos.
6. `kubectl apply -f deployment-live-stag.yaml` — picks up the pod-spec change and rolls automatically.
7. Verify per the steps below before calling it done.

## Protocol — Verify / Troubleshoot

```bash
# 1. Key reached the container? (prints length, never the key itself)
kubectl exec -n tgekyc-staging deploy/tgekyc-live-stag -- sh -c 'echo ${#OPENAI_API_KEY}'
# 0 = did NOT reach the container — check Secret exists in tgekyc-staging and name/key match the manifest exactly

# 2. Pod healthy
kubectl get pods -n tgekyc-staging -l app=tgekyc-live-stag
kubectl logs -n tgekyc-staging deploy/tgekyc-live-stag --tail=100

# 3. Real end-to-end test — run an actual liveness video through MPAY's doLivenessVideo flow
```

Read `reasoning` / `confidence_reason` in the response:

| Symptom | Cause |
|---|---|
| `"OpenAI API key not configured"` | Secret missing, wrong name/key, or not in `tgekyc-staging` namespace, or pod not restarted after Secret was created |
| `service_error: api_error` | Key present but the OpenAI call failed — invalid/expired key, no egress from the pod, proxy/NetworkPolicy blocking, rate limit |
| `service_error: no_frames` | Uploaded video couldn't be decoded — corrupt/empty/unsupported format |
| Pod `OOMKilled` | Memory limit too low for the video length/resolution, or too many concurrent requests on one pod — current limits are `10Gi` request / `20Gi` limit, still above the vendor's baseline recommendation of 6Gi, so this is unlikely to be the first cause to check |

---

## Mandatory Rules

1. **This is tgekyc-specific** — do not generalize this skill's steps to other vendors' K8s deployments; if a different eKYC/liveness vendor comes up, treat it as a new investigation, not this checklist
2. **Never assume `.env` reaches a K8s pod** — always check for a Secret + `secretKeyRef`, regardless of what the build folder's `.env` contains
3. **`SAVE_DATA` stays `"0"` unless Dejul explicitly says otherwise** — this is biometric data retention, a compliance-sensitive decision, not a default to flip casually
4. **Don't silently add probes back** — Dejul declined them once; ask before reintroducing, don't treat their absence as a bug to auto-fix
5. **Re-read the vendor's current `KUBERNETES.md` before every image upgrade** — assume requirements can change between versions, don't rely on this file's snapshot alone

---

## Edge Cases

| Situation | Behavior |
|---|---|
| `deployment.yaml` / `deployment-dev.yaml` / `deployment-staging.yaml` mentioned | These are the older tgekyc OCR service manifests, unrelated to the liveness image — confirm which manifest before editing anything |
| User asks to enable video retention | Confirm explicitly this is a deliberate compliance decision before flipping `SAVE_DATA` to `"1"` |
| Vendor sends a new `.env` file for an upgrade | Same trap as before — extract the value, put it in the `liveness-openai` Secret, never mount/copy `.env` into the pod |
| `echo ${#OPENAI_API_KEY}` returns non-zero but errors still occur | Key reached the container but may be invalid/expired/wrong project — check `service_error` field, not just presence |

---

## Level History

- **Lv.1** — Base: known config table, redeploy/upgrade protocol, verify/troubleshoot protocol, mandatory rules. (Origin: 2026-08-01, first deployment fix — `OPENAI_API_KEY` not reaching the pod because K8s doesn't read `.env`, `SAVE_DATA` disabled per Dejul, probes declined)
