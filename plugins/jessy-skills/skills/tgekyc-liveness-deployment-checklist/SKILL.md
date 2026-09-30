---
name: tgekyc-liveness-deployment-checklist
description: "MUST use when working on the tgekyc liveness service deployment —
             redeploying, upgrading the vendor image version, checking its K8s
             config, verifying the deployed version via /version.json, or
             diagnosing 'OpenAI API key not configured' /
             FAKE_FACE errors from MPAY's doLivenessVideo flow. Also triggers on:
             'tgekyc liveness', 'liveness deployment', 'check liveness config',
             'redeploy liveness', 'upgrade liveness image', 'tgekyc-live',
             'check liveness version', 'confirm deployed version', 'push latest
             version liveness tgekyc', 'build and push liveness image'."
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
| Image | `localhost:30445/tgekyc-liveness:<tag>` — tag changes on every redeploy, always read the current value from `deployment-live.yaml`, never hardcode/remember it (last known: `1.064`, 2026-09-30) |
| Registry, external push address | `10.5.1.43:30445` — same registry as `localhost:30445` above, just the address reachable from Dejul's workstation (confirmed via `docker push`/`curl .../v2/`); the manifest keeps `localhost:30445` since that's resolved from inside the cluster node |
| Vendor build source | `C:\PROJECTS\EKYC\Deployment\liveness_detection-v<version>-<YYYYMMDD>\` — vendor now drops a **new dated folder per version** (e.g. `liveness_detection-v1.3.2-20260929`), not updates to a shared `master` folder. Auto-detect the newest by date suffix; don't assume a fixed folder name |
| CPU (request / limit) | `2` / `4` cores — matches vendor's v1.3.1 spec (peaks ~3 cores during decode, settles ~2 while running; vendor recommends 2-4 per pod assuming 1 request at a time) |
| Memory (request / limit) | `3Gi` / `6Gi` (lowered from `10Gi`/`20Gi` 2026-08-03 per vendor spec — v1.3.1 uses ~1.8GiB at 5-6s/30fps, ~3.2GiB at 7s/60fps, ~4.9GiB hard max at ~600 frames) |
| Manifest path | `C:\PROJECTS\DOCKER GITLAB\docker\tgekyc\deployment-live.yaml` (renamed from `deployment-live-stag.yaml` 2026-08-03) |
| Vendor setup doc | `C:\Users\opera\OneDrive\Desktop\KUBERNETES.md` (Ctrl CV) |
| Cluster access | `kubectl` from this machine only has a `minikube` context configured; the real cluster (kubeconfig files `KVM1DC`/`KVM2DC`/`KVM7DC`/`KVMDR` in `C:\PROJECTS\DOCKER GITLAB\docker\kube\`) all point to `https://127.0.0.1:6443` — meaning an SSH tunnel to the cluster host must already be open before `kubectl apply` will work. No tunnel = no route. Never assume `kubectl` will reach `tgekyc` from a fresh shell; confirm with `kubectl get pods -n tgekyc` first, and if it fails, that's an environment gap, not a deployment bug |
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

## Protocol — Build & Push New Version

Trigger: "push latest version liveness tgekyc", "build and push liveness image", or similar.

1. **Find the source folder** — scan `C:\PROJECTS\EKYC\Deployment\` for folders matching `liveness_detection-v*-*`, pick the one with the newest date suffix. Don't reuse a stale local checkout.
2. **Determine current version** — read the `image:` tag in `C:\PROJECTS\DOCKER GITLAB\docker\tgekyc\deployment-live.yaml`. This is the source of truth, not the registry's tag list, not local `docker images`, and not this file's "last known" note above (which can go stale) — always re-read the manifest fresh.
3. **Compute the next version** — see Version Increment Rule below.
4. **Patch the Dockerfile if needed** — every vendor drop is a fresh folder with a fresh Dockerfile, so it will NOT carry forward the fix below. Check the `apt-get` step for the HTTPS-mirror switch + retry flags; add them if missing (see Known Issue).
5. **Build**: `docker build -t tgekyc-liveness:<next> .` from the source folder. Use `--progress=plain` if it fails, to see the actual per-package error rather than just the summary line.
6. **Tag for the registry**: `docker tag tgekyc-liveness:<next> 10.5.1.43:30445/tgekyc-liveness:<next>`
7. **Push**: `docker push 10.5.1.43:30445/tgekyc-liveness:<next>`. Large layers (opencv etc.) can time out (`net/http: timeout awaiting response headers`) on this network — retry the push; already-pushed layers report `Layer already exists` and are skipped, so retries are cheap. 2-3 attempts has always been enough; only treat it as a real problem if a *different* layer fails each time.
8. **Update the manifest** — bump `image:` in `deployment-live.yaml` to `localhost:30445/tgekyc-liveness:<next>` (keep the `localhost:30445` prefix in the manifest — that's resolved from inside the cluster node; `10.5.1.43:30445` above is only the address reachable from Dejul's workstation for the push).
9. **Stop — do not run `kubectl apply`.** This session has no route to the real cluster (see Cluster access in Known Configuration). Tell Dejul the image is built, pushed, and the manifest is updated with the new tag, and that he needs to run `kubectl apply -f deployment-live.yaml -n tgekyc` himself from wherever his tunnel is set up. Only attempt it yourself if `kubectl get pods -n tgekyc` already succeeds in the current shell (i.e., a tunnel happens to already be open) — never try to establish the tunnel yourself.
10. Once Dejul confirms he applied it, offer the Verify Deployed Version protocol below to confirm `system_version` actually changed.

### Version Increment Rule

The tag's fractional part is a running counter, incremented as `current + 0.001`, then printed with the fewest decimal digits that don't drop below 2 (i.e., drop a trailing zero only when it lands in the 3rd decimal place):

- `1.064` → `1.065` → `1.066` → … → `1.098` → `1.099`
- `1.099 + 0.001 = 1.100` → printed as **`1.10`** (trailing zero dropped, stays at 2 decimals — not `1.1`)
- `1.10` → `1.101` → `1.102` → … (3 digits resume once the hundredths digit is non-zero again)

Always compute this from the version actually found in `deployment-live.yaml` in step 2 — never guess or reuse a number remembered from an earlier session.

### Known Issue — apt-get build fails with 403 / Hash Sum mismatch / connection reset

`deb.debian.org` sits behind a Fastly CDN, and this network's plain-HTTP (port 80) path to it is unreliable — large `apt-get install` layers (e.g. `tesseract-ocr` + its dependency chain, ~50-60MB) intermittently fail partway with a mix of `403 Forbidden`, `Hash Sum mismatch`, and `Error reading from server. Remote end closed connection`, all from the same edge IP. `docker push` of large layers over this same network can time out the same way — see step 7 above.

**Fix, applied in the Dockerfile's apt-get RUN step** (confirmed working, 2026-09-30):
```dockerfile
RUN sed -i 's|http://deb.debian.org|https://deb.debian.org|g' /etc/apt/sources.list.d/debian.sources && \
    apt-get update -o Acquire::Retries=5 && \
    apt-get install -y --no-install-recommends -o Acquire::Retries=5 \
    <packages...> \
    && rm -rf /var/lib/apt/lists/*
```
This is a **local fix to this Dockerfile only** — the vendor's own Dockerfile in each new dated drop does not include it. Re-check/re-apply it every time a new vendor version folder is used, not just once.

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
8. **Never run or attempt to establish an SSH tunnel to the cluster yourself** — if `kubectl` can't reach `tgekyc`, that's Dejul's tunnel to set up, not something to work around
9. **Never guess the next version tag** — always read the current one fresh from `deployment-live.yaml` (step 2 of the Build & Push protocol) before computing the next
10. **Re-check/re-apply the apt-get HTTPS+retry fix on every new vendor drop** — it lives only in this locally-edited Dockerfile copy, not in anything the vendor ships

---

## Edge Cases

| Situation | Behavior |
|---|---|
| `deployment.yaml` / `deployment-dev.yaml` / `deployment-staging.yaml` mentioned | These are the older tgekyc OCR service manifests, unrelated to the liveness image — confirm which manifest before editing anything |
| User asks to enable video retention | Confirm explicitly this is a deliberate compliance decision before flipping `SAVE_DATA` to `"1"` |
| Vendor sends a new `.env` file for an upgrade | Same trap as before — extract the value, put it in the `liveness-openai` Secret, never mount/copy `.env` into the pod |
| `echo ${#OPENAI_API_KEY}` returns non-zero but errors still occur | Key reached the container but may be invalid/expired/wrong project — check `service_error` field, not just presence |
| Old commands/docs reference `tgekyc-staging` namespace or `tgekyc-live-stag` name | Stale — migrated 2026-08-03 to `tgekyc`/`tgekyc-live`. Update the reference, don't assume a second environment exists |
| `docker build` fails on `apt-get install` with 403/Hash Sum mismatch/connection reset | Known network flakiness to `deb.debian.org` — apply the HTTPS-mirror + retry fix (see Known Issue), don't assume the Dockerfile itself is broken |
| `docker push` times out on one specific layer | Retry the push a few times — already-pushed layers are skipped (`Layer already exists`); only escalate if a *different* layer fails on each attempt |
| `kubectl apply` requested but `kubectl get pods -n tgekyc` fails from this shell | No tunnel open — hand the apply step back to Dejul, don't try to open the tunnel yourself |
| New vendor version folder found (`liveness_detection-v<X>-<date>`) | Confirms vendor now ships dated folders per version, not a shared `master` — always pick the newest by date suffix |

---

## Level History

- **Lv.1** — Base: known config table, redeploy/upgrade protocol, verify/troubleshoot protocol, mandatory rules. (Origin: 2026-08-01, first deployment fix — `OPENAI_API_KEY` not reaching the pod because K8s doesn't read `.env`, `SAVE_DATA` disabled per Dejul, probes declined)
- **Lv.2** — Verify Deployed Version protocol: vendor-documented `/version` and `/version.json` endpoint (port 5010, no auth, never cached), expected response fields, in-cluster/port-forward/ingress call forms, `system_version` as the field to check. (Origin: 2026-08-03, vendor guide `C:\PROJECTS\EKYC\Deployment\how-to-call-version.md`)
- **Lv.3** — Production migration: renamed `tgekyc-staging`/`tgekyc-live-stag` → `tgekyc`/`tgekyc-live` (service was already running in production, old name was leftover); moved into the existing `tgekyc` namespace alongside the OCR deployment after confirming per-pod resource limits don't pool across a namespace; lowered memory to `3Gi`/`6Gi` (from `10Gi`/`20Gi`) per the vendor's v1.3.1 frame-count spec, CPU confirmed unchanged at `2`/`4`. Added the delete-old-Service-first migration sequence (NodePort reuse) as a one-time note. (Origin: 2026-08-03, client email with per-request memory/CPU breakdown)
- **Lv.4** — Build & Push New Version protocol: vendor now ships a dated folder per version (not a shared `master`), auto-detected by newest date suffix; version source-of-truth is `deployment-live.yaml`'s `image:` tag, not the registry or local `docker images`; version increment rule (`+0.001`, trim trailing zero at 3rd decimal only, e.g. `1.099`→`1.10`); registry has two reachable addresses (`localhost:30445` in-cluster, `10.5.1.43:30445` from Dejul's workstation); documented the `deb.debian.org` HTTPS+retry apt-get fix (403/Hash Sum mismatch/connection-reset over plain HTTP) as a per-vendor-drop fix, not a one-time one; documented that this session has no cluster route (kubeconfigs point to `127.0.0.1:6443`, need Dejul's SSH tunnel) — build/push/manifest-update only, `kubectl apply` always handed back to Dejul. (Origin: 2026-09-30, v1.3.2 build failure — apt-get 403s traced to Fastly edge instability; build fixed and verified end-to-end: built, pushed to registry, Dejul applied and confirmed working)
