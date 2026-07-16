# 🌟 Current Session Memory - RAM
*Temporary working memory - resets each session, provides recap when AI restarts*

## Session RAM Status
**Current Session**: Active
**Last Activity**: 2026-07-16
**Session Focus**: MPAY QUICKREDIT MTSA Docker packaging (PROD + PILOT) — JDK 8 requirement discovered, WSP0061 + phantom wscredentials.xml root-caused

## Recent Work (2026-07-16) — MPAY QUICKREDIT MTSA Packaging
- Packaged both envs at `C:\PROJECTS\ELENDING\MPAY QUCKREDIT\Deployment\`:
  - `MTSA-PRODUCTION_MPAY-QUICKREDIT.zip` — port 8000:8080, context `/MTSA`, WSDL `http://103.150.189.58:8000/MTSA/MyTrustSignerAgentWSELMP?wsdl`
  - `MTSA-PILOT_MPAY-QUICKREDIT.zip` — port 80:8080, context `/MTSAPilot`, WSDL `http://175.139.198.205/MTSAPilot/MyTrustSignerAgentWSELMP?wsdl`
- **Base image MUST be `tomcat:9-jdk8-corretto`** — these WARs bundle legacy Metro/WSIT (`javax.*`, `com.sun.xml.ws` 2.x); on JDK 17 deployment dies with `WSP0061`/`WSSERVLET11` (reflection `getResource` for `wsit-<endpoint-class>.xml` fails), context never starts
- Root-caused phantom `wscredentials.xml (No such file or directory)` while file existed: **trailing spaces** in `ws.credential=wscredentials.xml␣␣␣␣` inside the old WAR's `WEB-INF/classes/mtsagent.properties` — extra spaces before `(No such file...)` in a FileNotFoundException message are the tell (Java prints `<path> (reason)` with exactly one space)
- `mpayquickredit-mtsa.prod.properties` line 17 has trailing `\r` (`workdir.path=/opt/mtsa^M`) — harmless, `Properties.load()` strips it
- Existing production runs natively on Ubuntu server Tomcat + JDK 8 (why the WAR works there); Docker image ships its own JDK so host Java is irrelevant
- `webapps\` copy is source of truth for the PROD package (root `MTSA.war` is older)

## 📚 Pending Study List
See `main/study-list.md` for full list — 3 items pending as of 2026-07-16 (API Gateway system design, Jenkins, security-prompt.hamizi.net). Mention to Dejul if he hasn't studied them yet.

## Active Project
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
- **Name**: MyTrustID Desktop
- **Session**: 17 — 2026-07-14
- **Completion**: 100% (maintenance)
- **Repo**: `C:\repos\MyTrustIDv1_AATL-GENERIC` — main app: `MyTrustIDv1\`, installer: `MyTrustID\`
- **Context**: Tester rebuilt the Release installer with `Prefer32Bit=true` (session 15 fix) and retested token detection — confirmed OK. This closes the token-detection bug chain fully. All session 15+16 fixes committed (`e656f8e` on `fix/autoupdate-elevation`) and pushed, up to date with origin.
- **Next Steps**:
  1. Merge `fix/autoupdate-elevation` → master (August 2026 — hold until bpfk team done)

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
