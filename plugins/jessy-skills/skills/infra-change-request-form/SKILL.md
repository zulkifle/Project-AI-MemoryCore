---
name: infra-change-request-form
description: "MUST use when creating a Change Request Form (CRF) to ask the infra
             team to apply an infrastructure change — NFS/NAS mount for a project's
             config directory, firewall rule, DB access, cert renewal, or similar.
             Matches MSC Trustgate's Appendix B-4.0 CRF template. Also triggers on:
             'create CRF', 'CRF for [project]', 'change request form',
             'request mount', 'infra change request', 'request infra team'."
---

# Infra Change Request Form (CRF)
*Generates the 6 content fields of MSC Trustgate's official Appendix B-4.0 Change Request Form — ready to paste into the actual template/signing tool.*

## Activation

When this skill activates, output:

`📋 Infra Change Request Form — gathering details...`

Then execute the protocol below.

## Context Guard

| Context | Status |
|---------|--------|
| **User wants to request an infra change** (NAS mount, firewall, DB access, cert, etc.) | ACTIVE — full protocol |
| **User wants a NAS mount for a project's config directory** | ACTIVE — use the NAS Mount Convention below to draft the Purpose paragraph |
| **User wants to package/dockerize a project** | DORMANT — that's `mtsa-container-packaging`, not this |
| **User wants to actually apply the change (has infra/root access)** | DORMANT — this skill drafts the request document, not the change itself |
| **User wants the approval chain filled or the change executed** | DORMANT — Requested by/Endorsed by/Authorized by and Section 2 Follow-up are handled outside this skill (signed via Dejul's own signing platform, then executed by infra) |

---

## Known Conventions

### The Form Itself

MSC Trustgate's official **Appendix B-4.0 — Change Request Form** ("INTERNAL USE ONLY - Version 1.0") has exactly **6 content fields**, in this order:

1. **Purpose** — what's being changed and what it's for, in one paragraph. The technical detail (e.g. the mount path) is written *inline* here — there is no separate "description of change" field.
2. **Reason** — why now / what's driving the request (current pain point, migration, new deployment).
3. **Impact** — bullet list: expected downtime (usually "No expected service downtime" for additive changes), risk level, and the failure mode if the change can't be applied cleanly.
4. **Note** — anything else worth flagging; blank ("None") if nothing applies.
5. **Activities** — numbered step list, written as literal actions the infra team performs.
6. **Planning Date & Time** — `Date:`, `Time:`, `Estimated Duration:` — use `TBC` for date/time if not yet scheduled, but always give a duration estimate.

Below the content fields is an **approval chain** (Requested by → Endorsed by → Authorized by, each signed + dated via Dejul's signing platform) and a separate **Section 2: Follow-up Section** (Status, Completed by, Date, action plan if not completed, Validated by) filled in by infra *after* the change is executed. **This skill only drafts the 6 content fields** — the approval chain and follow-up section are outside its scope (see Context Guard).

### NAS Mount Request (first documented pattern)

Project config directories are served from `tgnas-dc` under a per-client shared folder, mounted onto the app's server(s):

```
tgnas-dc:/volume7/appshared/<client-folder>/<project>
```

**The mount target is usually "all nodes of the Kubernetes cluster," not a single server** — confirmed by the precedent below, where the app ran as pods across a 4-node cluster and needed every node to see the same config. Only assume a single-server target if the app actually runs on one host; ask if unsure.

| Placeholder | Meaning | Example |
|---|---|---|
| `<client-folder>` | Tenant/client folder under `appshared` | `quickcash` |
| `<project>` | Project name, same on both sides of the mount | `mtsa`, `mtidverchecker` |
| local path on the node | Where the app expects the file | `/opt/<project>/...` |

**Ask for `<client-folder>` and `<project>` — never guess the tenant folder name**, it doesn't always match the project name (e.g. `quickcash` serves both `mtsa` and `mtidverchecker`).

**Mount at the project root, not an environment-specific subfolder, if more than one environment could ever need it.** An NFS mount is a single mountpoint — mounting `.../mtsa` → `/opt/mtsa` means anything under `prod/`, `pilot/`, etc. on the NAS side just appears at `/opt/mtsa/prod`, `/opt/mtsa/pilot` with zero app-side path changes and no second CRF later. Mounting at `/opt/mtsa/prod` specifically works today but means adding PILOT later requires a second mount request — possibly touching a mount that's already live. Ask whether other environments are on the roadmap before picking the mount target; default to the project root if unsure and multiple environments are plausible.

**If a sibling environment is planned but has no files yet, create its empty folder on the NAS in the same Activities list** (e.g. an empty `pilot/` alongside a populated `prod/`) rather than treating it as a future, separate change — it's zero extra risk in the same mount activity and avoids a second CRF purely to create a folder. Always confirm whether the sibling environment already has real files to migrate (treat as its own backup/copy step) or is genuinely empty for now — don't assume either way.

**Default Activities to a whole-folder copy, not a per-file list, once the local source already contains the full structure to move.** If the environments (e.g. `prod/`, `pilot/`) already exist as subfolders under one local project directory (e.g. `/opt/mtsa`), Activities should read "backup/copy the `<project>` folder, including its `prod/` and `pilot/` subfolders" rather than enumerating individual filenames — simpler to read, and it naturally carries along anything already inside each subfolder (populated or empty) without the CRF needing to know every filename. Only enumerate individual files when there's no existing parent folder to point at (e.g. files scattered across different paths, or a single flat file rather than a folder tree).

### Precedent on Record: `mtidverchecker` NAS Mount (2026-07-14 → 2026-07-21)

The first real CRF filed using this pattern — reference this when drafting future mount requests for tone and level of detail:

> **Purpose**: Mount the NAS shared directory (`tgnas-dc:/volume7/appshared/quickcash/mtidverchecker`) to all Kubernetes nodes to centralize the `configv2.properties` file used by the MyTrustID Desktop Version Checker web service.
>
> **Reason**: The configuration file is currently stored locally on a single server (`/opt/mtidverchecker/mtid/configv2.properties`). Since the application runs on a 4-node Kubernetes cluster, centralizing the file on NAS ensures all pods use the same configuration and simplifies future updates.
>
> **Impact**: No expected service downtime. Low risk during the mount activity. If the NAS is unavailable, the application may be unable to read the configuration file.
>
> **Note**: (none)
>
> **Activities**:
> 1. Backup the existing configuration file.
> 2. Copy `configv2.properties` to the NAS shared directory.
> 3. Mount the NAS path on all Kubernetes nodes.
> 4. Verify the mount and application access to the configuration file.
> 5. Restart the affected pods if required and perform service verification.
>
> **Planning Date & Time**: Date: TBC · Time: TBC · Estimated Duration: 30–60 minutes

Requested by Dejul 2026-07-14, endorsed same day, authorized 2026-07-20, completed and validated 2026-07-21.

### Reason Variants Seen So Far

The "why" for a NAS mount tends to fall into one of these — ask which applies, and ask for the exact local path and file names before drafting, don't assume:

1. **Migrating off a single server's local file** (mtidverchecker precedent) — config currently lives at a local path on one host; centralizing it on NAS is what lets a multi-node cluster share it.
2. **Config pinned to one node's local disk, not centralized across the cluster** (mtsa, 2026-08-06) — files already exist locally on a specific node (e.g. `/opt/mtsa/prod` on node `kvm8`) but aren't mounted from shared/NAS storage on any node. This ties the pod to that one node — if it's rescheduled elsewhere, the config isn't there. Get the exact node name and file names; they go in Activities as concrete backup/copy steps, not a generic "configuration files" placeholder.

---

## Protocol

### Step 1: Gather Details

Display this intake form and wait for the user's answers:

```
╔══════════════════════════════════════════════════════════╗
║           CHANGE REQUEST FORM — INTAKE                   ║
╚══════════════════════════════════════════════════════════╝

  Change Type*          : [ ] NAS Mount   [ ] Other: ___
  Project / System*     : (e.g., mtsa)

  --- If NAS Mount ---
  Client/Tenant Folder* : (e.g., quickcash — under appshared/)
  Mount Scope*          : [ ] All nodes of K8s cluster: ___
                          [ ] Single server: ___
  Config file affected* : (e.g., configv2.properties — what's
                           being centralized/moved)
  Currently stored at*  : (existing local path, if migrating
                           off a single server like the
                           mtidverchecker precedent)

  --- All change types ---
  Reason*                : (what's driving the request — current
                            pain point, new deployment, migration)
  Downtime / Risk*        : (expected downtime, risk level, and
                            what breaks if the change can't be
                            applied cleanly)
  Note                    : (anything else — optional, "None" if
                            nothing applies)
  Activities*             : (step-by-step actions infra performs
                            — backup, copy/apply, mount/configure,
                            verify, restart-if-needed)
  Planning Date & Time*   : (Date/Time — TBC is fine — plus an
                            estimated duration)

══════════════════════════════════════════════════════════
  Fill in the fields above and I'll draft the CRF content.
══════════════════════════════════════════════════════════
```

- [ ] If Change Type is NAS Mount: draft the Purpose paragraph inline using the convention above (`Mount the NAS shared directory (<full tgnas-dc path>) to <scope> to <business reason in one clause>.`)
- [ ] If Change Type is Other: ask the user directly for the technical detail — no convention exists yet for other change types (see Mandatory Rules on documenting new patterns)
- [ ] Do not guess Reason/Downtime-Risk/Activities from the change type alone — draft a reasonable first pass if asked to, but mark anything not explicitly confirmed by the user

### Step 2: Generate the CRF Content

Output the 6 fields using the **Output Template** below. Do not save to a file — output the text directly for the user to paste into the actual Appendix B-4.0 template/signing tool, unless they explicitly ask for a file this time.

---

## Output Template

```
Purpose:
[One paragraph. For a NAS mount: "Mount the NAS shared directory
(tgnas-dc:/volume7/appshared/<client>/<project>) to <scope> to
<business reason>."]

Reason:
[What's driving this — current pain point, migration, new
deployment. If migrating off a single server, name the current
local path being replaced.]

Impact:
- [Expected downtime — usually "No expected service downtime" for
  an additive mount]
- [Risk level during the activity]
- [Failure mode if the change can't be applied cleanly, e.g. "If
  the NAS is unavailable, the application may be unable to read
  the configuration file."]

Note:
[Anything else worth flagging, or "None"]

Activities:
1. [Step 1 — e.g. backup existing config]
2. [Step 2 — e.g. copy/apply the change]
3. [Step 3 — e.g. mount/configure]
4. [Step 4 — verify]
5. [Step 5 — restart affected services if required, confirm]

Planning Date & Time:
Date: [TBC or actual date]
Time: [TBC or actual time]
Estimated Duration: [e.g. 30–60 minutes]
```

---

## Mandatory Rules

1. **Match the real form's structure — 6 fields, no invented sections.** Purpose carries the technical detail inline; there is no separate "Description of Change" field.
2. **Never guess the client/tenant folder name** — it doesn't always match the project name (`quickcash` serves both `mtsa` and `mtidverchecker`); always ask or confirm against the mtidverchecker precedent
3. **Default a NAS mount's scope to "all nodes of the K8s cluster,"** not a single server, unless the user confirms the app runs on one host — this is what the actual precedent did and why (multi-pod apps need every node to see the same config)
4. **Reason/Impact/Activities are the user's call** — draft a reasonable first pass from context if asked to, but flag anything not explicitly confirmed rather than presenting a guess as fact
5. **Don't save a file unless asked** — default output is inline text for copy-paste into the actual template/signing tool (2026-08-06 decision)
6. **Never draft the approval chain or Section 2 Follow-up** — Requested by/Endorsed by/Authorized by are signed via Dejul's own signing platform, and Section 2 is filled by infra after execution; this skill's job ends at the 6 content fields
7. **New change types get documented** — if a change type outside "NAS Mount" comes up (firewall rule, DB access, cert renewal, etc.), capture its convention in a new subsection under Known Conventions once a real precedent exists, same pattern as NAS Mount

---

## Edge Cases

| Situation | Behavior |
|---|---|
| User gives project name but not client/tenant folder | Ask — don't assume it matches the project name |
| App runs on a single server, not K8s | Use "single server" scope explicitly in Purpose — don't default to "all nodes" if it doesn't apply |
| Change type isn't NAS Mount and no convention exists yet | Ask the user for the full technical detail directly; note in the output that this is a new pattern with no established precedent |
| Migrating config off a single server (like mtidverchecker) | Name the exact current local path in Reason — that's what makes the "why now" concrete |
| Reason/Planning Date not yet known | Leave as `TBC` (matches the real form's own convention) rather than inventing a placeholder |
| User asks to fill approval names or Section 2 | Decline — explain those are handled via the signing platform / by infra after completion, not something to draft |

---

## Level History

- **Lv.1** — Base: 7-section CRF template with an invented "Description of Change" field, NFS Mount convention, inline-output-only default. (Origin: 2026-08-06, first request — mtsa NAS mount under quickcash)
- **Lv.2** — Corrected to match MSC Trustgate's real Appendix B-4.0 template after Dejul shared the actual filed form: dropped the invented section (Purpose carries the technical detail inline), scope corrected to default to "all K8s nodes" not a single server, added the real `mtidverchecker` precedent (2026-07-14 request → 2026-07-21 completed) as the reference example, explicitly scoped out the approval chain and Section 2 Follow-up as outside this skill. (Origin: 2026-08-06, same session — real form: `Change Request Form (CRF).pdf`)
