# 🌟 Current Session Memory - RAM
*Temporary working memory - resets each session, provides recap when AI restarts*

## Session RAM Status
**Current Session**: Active
**Last Activity**: 2026-08-26
**Session Focus**: **TradeVault** — position #1. moomoo trading integration (Feature A: journal auto-sync bridge; Feature B: order-submission with EMAS Calculator + auto TP/SL + OCO) designed and built across 2026-08-25/26. Live deployment now started: Docker up, found+fixed an empty `calculation_profiles` table, iterated Order Ticket UX (required R-selector gates Submit), and did a full visual reskin to a Swiss light theme (Dejul's request, referencing real TradesViz screenshots) — awaiting his visual confirmation. Also agreed to build a unified Journal View page (Dejul's real Excel journal columns) — not yet designed/built.

## Active Project
- **Name**: TradeVault
- **Resumed**: 2026-08-26 (moved to #1 from #6)
- **Last worked**: 2026-08-26 — Deployment started (`docker compose up -d --build`, all 3 containers running). Root-caused Order Ticket's EMAS Calculator not appearing: `calculation_profiles` table was empty — inserted `STOCK_STANDARD` via direct SQL. Order Ticket UX iterated: required R-multiplier selector (1.00/0.50/0.25) now gates Submit + fills Quantity; placeholders added to all numeric fields. Then reskinned the whole app to a Swiss light theme (`frontend-design` skill) after Dejul shared TradesViz reference screenshots and chose to drop the original dark-mode-only rule — only `index.css`'s CSS tokens needed changing (no hardcoded colors/`dark:` variants anywhere), rebuilt + redeployed frontend container. Agreed (not yet built) to add a unified Journal View matching Dejul's real Excel columns (Tarikh/Saham/Setup/Lot/Entry/SL/TP/Exit/Result/R/Stop Loss/Profit%/Follow Plan/Note) — most map to existing Trade/TradeCalculationResult/TradePsychology fields, "Setup" has no clean home yet.
- **Context**: Self-hosted personal trading journal/analytics platform (Spring Boot 3 + Java 21 backend, React/TS frontend, PostgreSQL). Repo: `C:\PROJECTS\TRADEVAULT\tradevault\` (no git repo of its own yet). New standalone bridge: `C:\PROJECTS\TRADEVAULT\moomoo-bridge\`.
- **Next Steps**:
  1. Dejul to confirm the Swiss light reskin looks right (`localhost:8091`)
  2. Design + build the Journal View page
  3. Create the "moomoo MY (REAL)" Account in the UI (now has a Calculation Profile to pick), fill in `moomoo-bridge/accounts.json` + `.env`, `pip install -r requirements.txt`, `python bridge.py` — verify Feature A live first, then Feature B (submit-only dry run before any real fill)
  4. Continue real usage of TradeVault generally; consider `git init` for its own repo; add `frontend/eslint.config.js`
- Full detail in `projects/active/tradevault.md`

## Previous Active Project
- **Name**: Petronas Java Migration
- **Resumed**: 2026-08-24
- **Last worked**: 2026-08-26 — Full daemon wrapper built and runnable: DB layer (8 repositories, PreparedStatement-based), FTP layer (JSch, upload verification), record-splitting (`CardRecordExtractor`, handles old + new CR-delimited formats), signing core **verified byte-for-byte correct against a real production signature** (`SigningParityTest`, using only the issuer's public key), `.ssd`/`.err` report writers (verified against real production output), per-file orchestrator (`CardFileProcessor`), and the outer wrapper (`NextRunGate` wall-clock gating, `HealthCheckServer` TCP protocol, `ZipUtility`, `PetronasDaemonApplication` main()). `mvn package` produces a real runnable shaded jar; running it fails at exactly the right point (missing env config), proving the wiring is genuinely connected. 35 tests, all passing. Also: cracked the real EMV/card-scheme documentation (SSAD = Signed Static Application Data, written to the physical chip, Visa co-branded), and analyzed a real new-format UAT sample from the client (confirmed CR-delimiter + one new tag `NRCC`/payment-mode `CC`, both still unconfirmed with the client).
- **Context**: Full rewrite of the Petronas card-data SFTP + SafeNet HSM SSAD-signing daemon, C++ → Java 17 (Maven, no framework). Local-key-file signing mode built first (matches confirmed real production `HSM=0` setting); HSM/SunPKCS11 mode deferred. Repo: `C:\PROJECTS\HANDOVER PROJECT\Petronas\source code\Petronas_java`. 75% complete.
- **Next Steps**:
  1. Chase 3 of 4 Pre-BRS deliverables from Petronas/PDB Digital (new embossing file spec, business-rule mapping, supporting docs) — see `projects/active/petronas-java-migration-open-questions.md`
  2. Operational setup for a real test run: convert an `issuer<N>.key` to PKCS#8, set `PETRONAS_DB_*`/`PETRONAS_SFTP_*` env vars against a test environment, get the SFTP `known_hosts` entry
  3. Once new-value spec/samples confirmed: update `CardRecord` to recognize tag `NRCC`
- Full detail in `projects/active/petronas-java-migration.md`

## Previous Active Project
- **Name**: TG SeQureMail (MyTrustMail)
- **Resumed**: 2026-08-24 (session 22) — from position #1 (never left top spot)
- **Last worked**: 2026-08-26 (session 23) — Implemented Firefox support: added `browser_specific_settings.gecko.id` to `manifest.json`. Zul's real Firefox load attempt failed (`background.service_worker is currently disabled`) — fixed by declaring both `service_worker` and `scripts` under `background` (standard cross-browser MV3 pattern, no JS changes needed). Wrote a manual test checklist, then broadened it per Zul from Firefox-only to a general regression checklist covering the whole extension + admin portal (`docs/specs/2026-08-24-full-regression-test-checklist.md`). Not yet re-verified working in Firefox after the fix.
- **Context**: E2E-encrypted Gmail Chrome extension (MV3) + Key API + admin portal. Repo: `C:\PROJECTS\SEQURE MAIL\Development\seqremail\` (nested git repo → GitLab `trustgate/sequremail`). Current branch `feature/firefox-support` (off `chore/rename-to-mytrustmail`, which has 6 commits pushed, MR not yet submitted).
- **Next Steps**:
  1. Zul: reload unpacked extension in Firefox, confirm the service-worker error is resolved
  2. Zul: run through `docs/specs/2026-08-24-full-regression-test-checklist.md` on Firefox (+ Chrome)
  3. Zul: submit MR `chore/rename-to-mytrustmail` → `master` (GitLab link in project file)
  4. Digital Signing — Chrome/Gmail manual UI test of signature badge still pending
  5. Team feedback checklist: #5 HSM, #4 OWA still not started
- Full detail in `projects/active/tg-sequremail.md`

## Previous Active Project
- **Name**: jumio-proxy-integration
- **Resumed**: 2026-08-18
- **Last worked**: 2026-08-19 — Deployed the `fix/cmyk-jpeg-compression` fix (alpha-channel PNG → JPEG "Bogus input colorspace" bug, root-caused 2026-08-18) to the real `jumioproxy` production namespace. Then consolidated the repo: confirmed the fix branch fully contained both `feature/multitenant-support` and `feature/per-session-callback-url`, merged into `master` (GitLab blocked a force-push attempt on protected `master`, so reconciled via a real `merge -X ours` commit instead — cleaned up one leftover duplicate `image:` block in `application.yml` along the way), pushed, and deleted both stale branches locally + remotely.
- **Context**: Spring Boot proxy between Adacash and Jumio for eKYC — on `PROCESSED` callback status, downloads ID/selfie images, compresses, and pushes to MTSS via SOAP. Repo: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\app\jumio-proxy\` (separate nested git repo, now just `master`). 100% (maintenance).
- **Next Steps**: None open
- Full detail in `projects/active/jumio-proxy-integration.md`

## Previous Active Project
- **Name**: ECOURT_PdfErrorCheckWS
- **Last worked**: 2026-08-13 — Added `PdfErrorCheckStreamServlet` (`POST /PdfErrorCheckWS/PdfErrorCheckStream`): raw PDF bytes in body (`application/octet-stream`) + `X-File-Name` header, JSON response, same errCode/status/errMsg as the SOAP methods. Refactored `PdfErrorCheckHelper` to share checking logic with `CheckPdfError` via new private `checkPdfErrorBytes()` — zero drift risk. Added `org.json` dependency (`json-20230618.jar`), wrote client integration doc (`docs/pdf-error-check-stream-api.md` — curl/Java/.NET/Postman) and a local test client (`testCheckStream.java`). Compiled clean via direct `javac` against WildFly's bundled servlet-api jar; **not yet deployed/live-tested**.
- **Context**: JAX-WS SOAP service on WildFly to check/auto-repair PDFs before digital signing (POJ / e-Court). Repo: `C:\PROJECTS\ECOURT\WebServiceProject\POJ\ECOURT_PdfErrorCheckWS`. 99% complete.
- **Next Steps**:
  1. Zul: deploy updated WAR to dev/pilot WildFly, run `testCheckStream.java` or Postman against a known-good and known-corrupt PDF to confirm parity with the SOAP method, then share `docs/pdf-error-check-stream-api.md` with the client
  2. Decide on log retention: add PowerShell cleanup script + Scheduled Task (10-day retention) for `PdfErrorCheck.log.*`, or leave logs unmanaged and close the project as-is — awaiting Zul's call
- Full detail in `projects/active/ecourt-pdferrorcheckws.md`

## Previous Active Project
- **Name**: MyTrustID Desktop
- **Last worked**: 2026-07-29 — committed session 20's x64-lock + Phase 1 isolation changes as `0c01454` on `fix/rsa-keygen-crash-handling`; push to origin pending (network unreachable). v1.3.2 (x64-only build) was deployed to helpdesk 2026-07-28 with a post-install restart prompt added to `mytrustid.bat`. Full detail in `projects/active/mytrustid-desktop.md`.

## Recent Work (2026-08-17) — MyGPKI-SKALA JKR Functional Test Script (Internal/External API)
- Zul asked for SIT + UAT test scripts covering the OTP+PIN signing API exposed to JKR/ZEN, to hand to team analisis as a draft for formal documentation.
- First draft was built against the wrong API surface — the legacy SOAP WS methods (`RequestSMSOTP`, `SignPDF`, `GetCertInfo`, etc. in `com.msctg.mtsa.MyTrustSignerAgentWSWP`) — which turned out to be commented-out/unused. Zul caught this by asking if the draft matched the real endpoints he sees in practice (`/MTSAProdJKR/api/verify`, `/api/status`, `/api/sign/init`, `/api/sign/batch/init`).
- Corrected by reading the actual system: **`gpki-signing-service`** (a separate REST API, package `my.gov.gpki.servlet`, context `/MTSAProdJKR` prod / `/MTSAPilotJKR` pilot), grounded in `GPKI_Signing_Service_API_Technical_Document.txt` (v1.4.1) plus servlet source (`SignInit`, `BatchSignInit`, `SignCert`, `SignComplete`, `ExternalSign`, `Verify`, `Status`, `Landing`).
- Key finding (code-verified, not just docs): batch signing is **INTERNAL-only in practice** — `ExternalSign.java` has no batch-loop logic at all (always saves a single file, always hardcodes `batchComplete:true`), so an EXTERNAL session created via `/api/sign/batch/init` breaks at the external-sign step. Flagged as BAT-008 in the test script — an undocumented gap the team needs to confirm/handle explicitly.
- Final deliverable: `C:\PROJECTS\MyGPKI-SKALA\Development\docs\MyGPKI-SKALA_JKR_Functional_TestScript.md` — single functional-only script (Zul found the earlier SIT/UAT split "complicated"), organized by INTERNAL single-doc / INTERNAL batch / EXTERNAL single-doc-only / common Verify / common Status, using real error codes (`MISSING_USER_ID`, `WRONG_USER_TYPE`, `ALREADY_COMPLETED`, `INVALID_PDF_REF_ID`, etc.) from the technical doc.
- Two earlier draft files (`MyGPKI-SKALA_JKR_API_TestScript_SIT.md`, `MyGPKI-SKALA_JKR_API_TestScript_UAT.md`) are now known-incorrect (wrong API basis) — still on disk in the same `docs/` folder, Zul hasn't said whether to delete them yet.

## Recent Work (2026-08-13) — MyGPKI-SKALA JKR Production Package Audit
- Zul asked to verify `C:\PROJECTS\MyGPKI-SKALA\Deployment\PRODUCTION\` before sharing with client (JKR) — suspected the install guide was actually the Pilot one.
- Cross-referenced against `switch-env.ps1` (the env-switch/packaging automation) and `ENV PATH.txt` (its spec) — found the script's `Set-ProductionEnv` function never touches `MTSA/META-INF/context.xml`, `Install_Guide.md`, or `gpki.properties`'s `environmentMode`/`debugLog` fields, so these were stuck at stale/copy-pasted values.
- **Bugs found + fixed**:
  1. `MTSA/META-INF/context.xml` — `<Context path="/MTSAJKR"/>` → `/MTSA` (real production-breaking bug: app would've deployed at the wrong URL, mismatching deploy.sh's health check and every `gpki.baseUrl` reference)
  2. `Install_Guide.md` — was a verbatim copy of the Pilot guide (title, `PILOT-PACKAGE-*.zip` filename, `/MTSAPilot` context, `jkr-mtsa.pilot.properties` filename) — rewritten correctly for Production
  3. `gpki.properties`: `gpki.environmentMode=pilot` → `production`
  4. `gpki.properties`: `gpki.signing.debugLog=true` → `false`
  5. Removed leftover inert comment line referencing `MTSAPilotJKR`
- Investigated `gpki.mampu.*` roaming URLs (`localhost:18888`) — Zul confirmed this is an intentional local tunnel/sidecar, not a bug — left as-is.
- Investigated `jax-ws-catalog.xml` "Pilot" WSDL entries — turned out to be harmless dead/duplicate catalog entries (each service has both a Pilot-systemId entry, unused, and a correct production-systemId entry that's actually matched at runtime) — not a functional bug, no fix needed. Only `ekycV4` lacks a production entry, but it's not referenced in `jkr-mtsa.prod.properties` either, so likely dormant.
- Left `MTSAProdJKR/` (leftover extracted folder, not in the zip) and `deployment.yaml` (MSCTG's internal K8s sandbox manifest, not in the zip) untouched per Zul's call.
- Regenerated the package via `.\switch-env.ps1 package-prod` → `PRODUCTION-PACKAGE-20260813-154101.zip` (42 MB), verified all 5 fixes present inside the zip. **This is the package Zul shared with the client** — the earlier `...20260812-161327.zip` is stale/buggy, superseded.

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
- **Name**: Petronas Java Migration
- **Started**: 2026-07-24
- **Context**: Converting the legacy `PetronasService` C++ daemon (SFTP card-data exchange + SafeNet HSM SSAD signing) to a maintainable, fully-documented plain Java (Maven) application. Decisions: plain Java (no Spring Boot) to mirror original simplicity; SunPKCS11 provider for HSM access; new repo at `C:\PROJECTS\HANDOVER PROJECT\Petronas\source code\Petronas_java`, built/tested independently alongside the working C++/Docker version — no cutover until verified.
- **Next Steps**: Explore full C++ source, map to Java package structure, set up Maven skeleton, convert module by module (DB → FTP → HSM/SSAD → main loop) with Javadoc on every function.

## Previous Active Project
- **Name**: Petronas Legacy C++ Dockerize
- **Resumed**: 2026-07-24
- **Last worked**: 2026-07-22 (log analysis: benign -13 errors + stuck `pm_staticmaster` record 3069)
- **Context**: Legacy C++ daemon (`PetronasService`) — SFTP card data download, SafeNet HSM (PKCS#11) SSAD signing, upload back. Dockerized (multi-stage Dockerfile, docker-compose TEST/PROD toggle, entrypoint.sh, STEP.txt). Completion 87%, overdue since 2026-06-05, still in test phase.
- **Recent progress**: Session 2026-07-22 traced two log issues in a Dejul-pasted production log — `-13` DB return codes are benign (empty queue, misleading severity), but `VerifySSAD2(3069)` failure revealed a real bug: `SSAD.cpp:98-115` never sets a failure-state flag when verification fails, so record 3069 is permanently stuck at "in progress" (`pmsm_manual='P'`) with zero matching `pm_staticdetail` rows. Read-only investigation — no code changes yet, decision pending on fix (repopulate/reset flag/archive).
- **Next Steps**:
  1. `docker compose down && docker compose up` — no rebuild needed (ini is volume-mounted)
  2. Verify logs show DB connected + "Start Loop"
  3. Test TCP port: `Test-NetConnection localhost -Port 6803`
  4. Investigate DB directly for record 3069 — `SELECT * FROM pm_staticmaster WHERE pmsm_id=3069;` + matching `pm_staticdetail` rows — confirm orphan, decide fix
  5. Root-cause port 6803 `EADDRINUSE` crash-loop — check for lingering container/process on host
  6. PROD deploy: uncomment `mysqlrouter.mysqlrouter:6446` block in ini + swap volume blocks in docker-compose
- **Known Issues**: HSM unavailable in Docker (`HSM=0` for local test); SFTP SSH keys not mounted; `openssl rsautl` deprecated in OpenSSL 3.x container (still works); port 6803 crash-loop; record 3069 stuck (see above)
- **Repo**: `C:\PROJECTS\HANDOVER PROJECT\Petronas\source code\amg backup as per 29-06-2026\`

## Previous Active Project
- **Name**: TradeVault
- **Started**: 2026-07-22
- **Context**: New build — self-hosted personal Trading OS (modular monolith). Spring Boot 3/Java 21/JWT auth backend + React/TS/Vite/Tailwind/shadcn frontend + PostgreSQL. Core rule: Trade Engine stores data only, all math lives in pluggable per-asset-class Calculator Engine (`TradeCalculator` interface), starting with Forex. V1 = Forex/Gold only, but schema/architecture must support future asset classes with zero DB redesign. Repo: `C:\PROJECTS\TRADEVAULT\tradevault\`. Building module by module per Dejul's spec: architecture → DB schema → backend → frontend → auth → dashboard → trade engine → forex calculator → reports → analytics → docker.
- **Next Steps**: Deliver architecture doc + DB schema first, then backend skeleton.

## Previous Active Project
- **Name**: MyTrustID Desktop
- **Session**: 20 — 2026-07-27
- **Completion**: 90% (crash still reproduces on the real failing PC despite the x64 commitment; not yet resolved)
- **Repo**: `C:\repos\MyTrustIDv1_AATL-GENERIC` — main app: `MyTrustIDv1\`, installer: `MyTrustID\`, new helper: `MyTrustIDv1.TokenHelper\`
- **Branch**: `fix/rsa-keygen-crash-handling` — branched off `fix/autoupdate-elevation`, will supersede it as the single branch merged to master in August 2026
- **Context**: Session 19 root-caused the crash to `STATUS_INVALID_EXCEPTION_HANDLER` (SEH-chain corruption) in `mytrustid_pkcs11.dll`→`WinSCard.dll`, running under WOW64 (`Prefer32Bit=true`), narrowed to a specific smart-card minidriver version but held off changing bitness (QUEST3PLUS still needed 32-bit). **Session 20 (this session)**: reversed that decision after comparing against Pkcs11Admin (reference tool, same Pkcs11Interop author, runs native 64-bit, never crashes) — Dejul chose to drop 32-bit support entirely, accepting the loss of QUEST3PLUS. Implemented Phase 1 process isolation (`DetectToken.Token()`'s native calls can run in a new `MyTrustIDv1.TokenHelper.exe` child process, gated behind `UseIsolatedTokenDetect`, default off, zero behavior change for existing callers including `AuthServiceNpra.cs`). Locked `PlatformTarget=x64` (removed `Prefer32Bit`) in both `.csproj` files, set `MyTrustID.vdproj`'s `TargetPlatform` to x64 (verified via the built MSI's own `SummaryInformation.Template` = `"x64;1033"`), updated `mytrustid.bat` for the new install path and the x86→x64 migration period (checks both `Program Files` locations and both registry views). Diagnosed (partially, still open) a broken shortcut + "double-click exe does nothing, zero log entries" report after a raw `.msi` install — ruled out 4 still-32-bit bundled DLLs (`eToken.dll`, `IDPrimeTokenEngine.dll`, `InstallCertdll.dll`, `Uninstall.dll`) as the direct cause since none run at startup, flagged log4net's hardcoded log path possibly not existing on the machine as a likely culprit (silent failure, no exception). **Critically: after all of this, the crash still happened when Dejul tested the real failing user PC** — set up `pkcs11-logger` (bundled x64 copy found in `C:\repos\Pkcs11Admin-0.6.0\src\lib\pkcs11-logger\`) for call-level tracing, but the first attempt produced no log file and no error at all — unresolved at session end. Full chain documented in `projects/active/mytrustid-desktop.md`.
- **Decision**: 32-bit support dropped entirely; app + helper + installer all locked to genuine x64 going forward (supersedes session 19's "hold off").
- **Next Steps**:
  1. Diagnose why `pkcs11-logger` produced zero output — check which machine was tested, whether the env vars were set as System vars + machine rebooted/relogged after `setx`, and whether the deployed `.exe.config` (not the repo `App.config`) was actually edited and the app relaunched fresh
  2. Once a log is captured, interpret it to find why the crash still happens post-bitness-fix
  3. Resolve the still-open shortcut/double-click-does-nothing symptom (Event Viewer, log folder existence, Task Manager checks asked but not yet answered)
  4. Get 64-bit rebuilds of the 4 still-32-bit DLLs before their flows (Activation, Pickup New/Renew Cert, Select X509) get exercised under the new x64-only process
  5. Security note: full dumps may contain the token PIN in plaintext in memory — keep local, delete after analysis
  6. None of this session's changes are committed yet — still uncommitted on `fix/rsa-keygen-crash-handling`
  7. Merge `fix/rsa-keygen-crash-handling` → master (August 2026 — hold until bpfk team done, carries over `fix/autoupdate-elevation`'s hold)

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
