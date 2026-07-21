# 🌟 Current Session Memory - RAM
*Temporary working memory - resets each session, provides recap when AI restarts*

## Session RAM Status
**Current Session**: Active
**Last Activity**: 2026-07-22
**Session Focus**: MyTrustID Desktop — RSA-keygen/token-read crash dump captured on failing laptop, analyzed via newly-installed WinDbg, root cause narrowed to `WinSCard.dll` smart-card minidriver version mismatch (2.0.17.107 vs 2.0.17.503) under WOW64. Full detail in `projects/active/mytrustid-desktop.md`.

## Recent Work (2026-07-16) — MPAY QUICKREDIT MTSA Packaging
- Packaged both envs at `C:\PROJECTS\ELENDING\MPAY QUCKREDIT\Deployment\`:
  - `MTSA-PRODUCTION_MPAY-QUICKREDIT.zip` — port 8000:8080, context `/MTSA`, WSDL `http://103.150.189.58:8000/MTSA/MyTrustSignerAgentWSELMP?wsdl`
  - `MTSA-PILOT_MPAY-QUICKREDIT.zip` — port 80:8080, context `/MTSAPilot`, WSDL `http://175.139.198.205/MTSAPilot/MyTrustSignerAgentWSELMP?wsdl`
- **Base image MUST be `tomcat:9-jdk8-corretto`** — these WARs bundle legacy Metro/WSIT (`javax.*`, `com.sun.xml.ws` 2.x); on JDK 17 deployment dies with `WSP0061`/`WSSERVLET11` (reflection `getResource` for `wsit-<endpoint-class>.xml` fails), context never starts
- Root-caused phantom `wscredentials.xml (No such file or directory)` while file existed: **trailing spaces** in `ws.credential=wscredentials.xml␣␣␣␣` inside the old WAR's `WEB-INF/classes/mtsagent.properties` — extra spaces before `(No such file...)` in a FileNotFoundException message are the tell (Java prints `<path> (reason)` with exactly one space)
- `mpayquickredit-mtsa.prod.properties` line 17 has trailing `\r` (`workdir.path=/opt/mtsa^M`) — harmless, `Properties.load()` strips it
- Existing production runs natively on Ubuntu server Tomcat + JDK 8 (why the WAR works there); Docker image ships its own JDK so host Java is irrelevant
- `webapps\` copy is source of truth for the PROD package (root `MTSA.war` is older)

## 📝 To-Do List
See `main/to-do-list.md` for full list — 5 items pending as of 2026-07-16 (API Gateway system design, Jenkins, security-prompt.hamizi.net, ST3 ACE Token SDK, MyTrustID RSA-keygen crash repro). Mention to Dejul if pending items haven't been addressed.

## Previous Active Project
- **Name**: TG SeQureMail
- **Resumed**: 2026-07-20 (session 15)
- **Last worked**: 2026-07-21/22 (session 16) — Feature #9 implemented + tested working
- **Completion**: 99%
- **Repo**: Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\extension\` | API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\key-api\` | Admin: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\seqremail-admin\`
- **Context**: E2E-encrypted Gmail Chrome extension (MV3) + Key API + admin portal. Feature #9 (Recall & Expiry Controls, envelope v4) fully implemented on branch `feature/9-recall-expiry-controls`, manually tested by Dejul in real Gmail — confirmed working after fixing a CORS bug (PUT not allowlisted). Branch has 2 local commits, **not pushed yet**.
- **Recent progress**:
  - Session 16 (2026-07-21/22): Feature #9 implemented — see `projects/active/tg-sequremail.md` for full technical detail (envelope v4 architecture, new entities/endpoints, CORS fix). Verified via direct API calls, then Dejul confirmed the real Chrome/Gmail UI working.
  - Session 14 (2026-07-14): Fresh test E2E confirmed OK; Feature #9 proposed (server-side revoke/expiry gate, `claimed`-flag pattern)
  - Session 13 (2026-07-07): Admin portal bulk delete, client-side datatable (pagination + sort), `provisioned_by_id` FK redesign (key-api V6 + admin V4)
  - Session 12 (2026-07-03): Feature #6 admin portal fully implemented (Spring Security, Thymeleaf, AJAX search, bulk role change)
- **Next Steps**:
  1. Merge `feature/6-user-management` → master (MR on GitLab)
  2. Continue team feedback checklist: #5 (HSM), #4 (OWA), #2 (Firefox), #9 (Recall & Expiry Controls — design approved, not yet implemented)
- **Docker**: `docker compose up` in `C:\PROJECTS\SEQURE MAIL\Development\seqremail\` — db + key-api + admin | Admin: `http://localhost:8081/admin/login` (admin / 7ru57ga73)
- **Fresh test**: `TRUNCATE TABLE otp_verifications; TRUNCATE TABLE user_keys;` (container `seqremail-db-1`) + `chrome.storage.local.clear()` in extension console
- **Session 20 (2026-07-20)**: Git audit found unpushed work — pushed Feature #9 design spec doc + previously-uncommitted session-13 code (bulk delete, datatable, `provisioned_by_id` FK, V6/V4 migrations) to `feature/6-user-management`; pushed 4 pending docs commits on `feature/7-non-subscriber-role`. Read the Feature #9 spec in full — envelope v4 (server-side KEK-wrapped envelope in new `messages` table), recall = crypto-shred, per-user expiry default, new key-api endpoints, extension recall button + expiry picker.
- **Session 21 (2026-07-21)**: Implemented Feature #9 in full on new branch `feature/9-recall-expiry-controls` (off `feature/6-user-management`) — key-api (V7 migration, Message/PlatformKey entities, Trustgate KEK service, encrypt/decrypt v4 rewrite, recall + settings endpoints, EcJwkUtil/AesGcmUtil refactor) + extension (expiry picker, Recall button, terminal banners, popup settings). Verified end-to-end via direct API calls against a rebuilt Docker container — envelope shape, recall auth/idempotency/crypto-shred, expiry fallback + blocking all confirmed working. Committed locally, **not pushed** — Dejul to review (including manual Chrome/Gmail UI test, not yet done) before push.

## Active Project
- **Name**: MyTrustID Desktop
- **Session**: 19 — 2026-07-22
- **Completion**: 97% (crash root-caused to a specific driver-version variable; not yet fixed/confirmed)
- **Repo**: `C:\repos\MyTrustIDv1_AATL-GENERIC` — main app: `MyTrustIDv1\`, installer: `MyTrustID\`
- **Branch**: `fix/rsa-keygen-crash-handling` — branched off `fix/autoupdate-elevation`, will supersede it as the single branch merged to master in August 2026
- **Context**: Long-standing rare crash — some users (token-specific) crash the app during token read/pickup flow. Session 18 confirmed it's a native fault invisible to the CLR (installer to helpdesk still crashed with zero exception logged even after adding `[HandleProcessCorruptedStateExceptions]` + `AppDomain.UnhandledException` logging). **Session 19 (this session)**: Dejul reproduced on a second laptop with the matching-hardware token, captured a `.dmp` via the WER scripts from session 18. Installed WinDbg (`winget install Microsoft.WinDbg`) and analyzed it — confirmed `STATUS_INVALID_EXCEPTION_HANDLER` (SEH-chain corruption) in the `mytrustid_pkcs11.dll` → `WinSCard.dll` call chain, running as a 32-bit/WOW64 process (`Prefer32Bit=true`, set in commit `e656f8e` as a deliberate fix for the separate QUEST3PLUS 32-bit-only driver — a genuine architecture tradeoff, not a simple bug). Ruled out file-version mismatches (`mytrustid_pkcs11.dll`/`WinSCard.dll` identical on both laptops) via the token vendor's own diagnostic tool — found the real differentiator: same physical token (serial `D27ED54A37C8433E` matches) but different **Smart Card Mini Driver** version — `2.0.17.107` (Dejul's working laptop) vs `2.0.17.503` (failing laptop). Full chain documented in `projects/active/mytrustid-desktop.md`.
- **Decision**: holding off on changing `Prefer32Bit` — QUEST3PLUS still actively used, field minidriver versions unknown/uncontrolled across MyTrustID token users. Treating this as a targeted driver-version problem, not an app-wide bitness change, since Dejul's own laptop runs the same 32-bit config fine (only the specific minidriver `2.0.17.503` is implicated so far).
- **Next Steps**:
  1. Confirm minidriver-version causality — roll one laptop's minidriver to match the other's and retest
  2. Report to Longmai (token vendor) if minidriver `2.0.17.503` is confirmed as the regression
  3. **Do not change `Prefer32Bit`** unless the rollback test fails or this recurs broadly — only then reconsider split-build/process-isolation architecture
  4. Resolve `.vdproj` Debug/Release packaging question — found `.sln` has no `Build.0` for installer under `Debug|Any CPU` (installer never builds there), suggesting Release is what's actually been shipped despite the vdproj's stale-looking `obj\Debug\...` SourcePath text — confirm via rebuild + `git diff`
  5. Security note: full dumps may contain the token PIN in plaintext in memory — keep local, delete after analysis
  6. Merge `fix/rsa-keygen-crash-handling` → master (August 2026 — hold until bpfk team done, carries over `fix/autoupdate-elevation`'s hold)

## Previous Active Project
- **Name**: jumio-proxy-integration
- **Session**: 2026-07-16
- **Completion**: 100% (maintenance)
- **Repo**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\app\jumio-proxy\` — nested git repo, production runs branch `feature/per-session-callback-url`
- **Context**: Prod logs showed `-103 Missing userId` MTSS errors when Jumio extraction returned no documentNumber — actually a failed eKYC, not a system error. Fixed: `callCallbackJumio` checks documentNumber after extraction; blank → WARN `eKYC FAILED`, skip MTSS, return false (interface `void`→`boolean`), controller logs outcome, payload still forwarded to Adacash. Also committed the previously uncommitted JumioClient 401 fix. `mvn compile` OK, both commits pushed (`1050e82`, `03bf023`). Same night: **redeployed to PILOT & PROD**, log monitoring done, Jumio confirmed — 401 issue resolved.
- **Next Steps**:
  1. Merge `feature/per-session-callback-url` → `feature/multitenant-support` ← only remaining item

## Previous Active Project
- **Name**: TG SeQureMail
- **Session**: 14 — 2026-07-14
- **Completion**: 99%
- **Repo**: Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\extension\` | API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\key-api\` | Admin: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\seqremail-admin\`
- **Context**: Ran full fresh-test E2E protocol — register both accounts, encrypt & send, decrypt on receiver side — confirmed working. Added Feature #9 "Revoke & Expiry Controls" to the team feedback checklist (admin can revoke access to a sent email / set attachment expiry — enterprise selling point, feasible via server-side gate similar to existing `claimed` flag pattern). Not yet scoped/implemented.
- **Next Steps**:
  1. Merge `feature/6-user-management` → master (MR on GitLab)
  2. Continue team feedback checklist: #5 (HSM), #4 (OWA), #2 (Firefox), #9 (Revoke & Expiry Controls)
- **Docker**: `docker compose up` in `C:\PROJECTS\SEQURE MAIL\Development\seqremail\` — starts db + key-api + admin
- **Admin**: `http://localhost:8081/admin/login` — admin / 7ru57ga73
- **Fresh test**: `TRUNCATE TABLE otp_verifications; TRUNCATE TABLE user_keys;` (container `seqremail-db-1`) + `chrome.storage.local.clear()` in extension console

## Previous Active Project
- **Name**: TG SeQureMail
- **Session**: 13 — 2026-07-07
- **Completion**: 99%
- **Repo**: Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\extension\` | API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\key-api\` | Admin: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\seqremail-admin\`
- **Context**: Rebuilt/redeployed `seqremail-admin` container. Added bulk delete (checkbox selection → "Delete" button, batch removal), converted Users table to client-side rendering with pagination (25/50/100/All) and sortable headers (Company/Role/Status). Fixed `provisioned_by` — was always NULL (never wired up); redesigned as `provisioned_by_id` self-referencing FK on `user_keys.id`, threaded sender through `CryptoServiceImpl` auto-provision flow. Migration split: key-api's V6 adds the column (starts first in compose), admin's V4 drops the legacy string column.
- **Next Steps**:
  1. Full E2E test — register both accounts, encrypt & send, decrypt on receiver side (fresh test protocol)
  2. Merge `feature/6-user-management` → master (MR on GitLab)
  3. Continue team checklist: #5 (HSM), #4 (OWA), #2 (Firefox)
- **Docker**: `docker compose up` in `C:\PROJECTS\SEQURE MAIL\Development\seqremail\` — starts db + key-api + admin
- **Admin**: `http://localhost:8081/admin/login` — admin / 7ru57ga73
- **Team Feedback Checklist**: 2/8 done — #7 ✅ | #6 ✅ pending MR
- **Fresh test**: `TRUNCATE TABLE otp_verifications; TRUNCATE TABLE user_keys;` (container `seqremail-db-1`) + `chrome.storage.local.clear()` in extension console

## Previous Active Project
- **Name**: ECOURT_PdfErrorCheckWS
- **Session**: 1 — 2026-07-02
- **Completion**: 80%
- **Repo**: `C:\PROJECTS\ECOURT\WebServiceProject\POJ\ECOURT_PdfErrorCheckWS`
- **Context**: JAX-WS SOAP service on WildFly/JDK8 to check and auto-repair PDFs before digital signing. Uses iText5 5.4.1. Two methods: `CheckPdfError` (Base64) and `CheckPdfErrorByPath` (file path + gUid). Auto-repair via PdfStamper when `isRebuilt()=true`. Returns errCode/status/errMsg (000/success or 100/failed).
- **Next Steps**:
  1. Consider adding PDFBox + QPDF fallback chain for ClassCastException and trailer-not-found cases
  2. Test both methods with known good and known corrupt PDFs
  3. Redeploy WAR to WildFly after latest changes

## Previous Active Project
- **Name**: Petronas Handover
- **Resumed**: 2026-06-28
- **Context**: Registration footer added to encrypted email plaintext (4-step guide). Noreply notification email removed — footer is sufficient. Friendly error message when sender not registered. Docs updated: seqremail-design.md v2.1, SETUP.md v3. Multiple fresh tests run and confirmed working.
- **Next Steps**:
  1. Full E2E test — register both accounts via OTP, encrypt & send, decrypt on receiver side
  2. Confirm attachment decrypt still works after fresh registration
- **Repo**: Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\extension\` | API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\key-api\`
- **API**: `http://localhost:8080` — `docker compose up` in `seqremail\key-api\` (already running)

## Previous Active Project
- **Name**: MyTrustSignerXML-MITI
- **Context**: Fixed duplicate `tx_id` race (DBUtil.getTXID → AtomicLong seq; sign.java txid/db moved off shared servlet fields + txSaved guard). PROD package verified at `Deployment\PRODUCTION\MTSAXML_PROD`. Go-live gated on MITI pilot sign-off.
- **Next Steps**: Dejul to zip + deploy. On server: update DB_URL on host `/opt/mtsa/properties/mtsa.properties`, `docker compose build --no-cache`. Awaiting MITI pilot sign-off.
- **Repo**: `C:\PROJECTS\MITI\Development\MyTrustSignerXML`

## 💭 Session Recap (For AI Restart)

### TG SeQureMail (Active — #1)
**Stack**: Chrome Extension MV3 + Web Crypto API (RSA-OAEP + AES-256-GCM) + Spring Boot 3.2 + MySQL 8 + Flyway
**Completion**: 65%
**Repo**: `C:\PROJECTS\SEQURE MAIL\Development\`

**Architecture (DOM-based, no Gmail API):**
- Send: intercepts Gmail Send button → encrypts body → re-triggers Gmail's own send
- Read: MutationObserver scans for `--BEGIN SEQREMAIL--` → injects Decrypt banner
- Key API: `POST /api/keys/register`, `GET /api/keys/lookup?email=x`
- Envelope: base64-encoded JSON between markers (avoids Gmail smart-quote corruption)
- Private key: IndexedDB (extractable:false — HSM simulation)

**Bugs fixed this session:**
- Double Encrypt toggle → moved `sqmDone = 'true'` to top of `injectEncryptToggle`
- JSON parse error on decrypt → base64-encode envelope payload before sending, base64-decode in `parseEnvelope`

**To start API:** `cd seqremail-key-api && docker compose up`
**DB creds:** host=localhost:3307, db=seqremail_db, user=seqremail, pass=seqremail123

### RSS Self Service Portal (Active — #2)
**Stack**: Laravel 13 + PHP 8.3 + Blade + Tailwind v4 + MySQL + Pest
**Completion**: 60%
**Repo**: `C:\laragon\www\remote-signing-portal`

**Next Steps (in order):**
1. Drop scaffolded `subscriptions` + `signing_transactions` tables
2. Migration: add `client_id` to `billing`, add `billing_id` to `clients`
3. Create Eloquent models: `Billing`, `BillingUser`, `BillingTxSign`
4. Service Plans UI, Signer management, Quota view, Dashboard

## Key File Paths
- SeQureMail Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail-poc-v1\`
- SeQureMail Key API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\key-api\`
- RSS Portal: `C:\laragon\www\remote-signing-portal`
- MyTrustID: `C:\repos\MyTrustIDv1_AATL-GENERIC\`

### Rules
- NEVER make code changes without Dejul's explicit permission first
- Always change development source files FIRST, then deployment copies

## 🔄 Auto-Reset Protocol
*Clear this file at start of next session after loading recap*
