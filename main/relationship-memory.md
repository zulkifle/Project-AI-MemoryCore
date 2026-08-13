# 🤝 Relationship Memory - Understanding Dejul
*Learning your preferences, style, and needs*

## User Profile
- **Name**: Dejul (prefers to be addressed as **Zul** in conversation — confirmed 2026-08-13)
- **Relationship Style**: Friendly, collaborative partnership with Jessy
- **Communication Preference**: Casual, direct, asks "why" behind things
- **Primary Focus Areas**: Java, Spring Boot REST API, OOP design patterns, DevOps, PDF Digital Signing (MyGPKI/SKALA)
- **Goals & Priorities**: Build career-ready Java skills with best practices; maintain and develop MTSA signing platform
- **Tools**: NetBeans (IDE), Postman (API testing), Java 17, Maven
- **Experience**: Software Developer + DevOps; works on government PKI signing systems (MyGPKI, SKALA, MTSA)

## Communication Patterns

### Preferred Communication Style

**Settings (observed)**:
- **Tone**: Professional yet warm, casual
- **Detail Level**: Concise — prefers summaries and tables over walls of text
- **Response Length**: Short and direct; asks follow-up when needed
- **Energy Level**: Focused, task-oriented during work sessions
- **Formality**: Low — casual commands like "yes", "fix it", "save"
- **Language**: Bahasa Melayu for casual chat/banter in the conversation itself, English for anything technical (code, logs, file names, commit messages, error terms). Confirmed 2026-08-11 — Dejul said full English gives him a headache ("pening"). Applies to all future sessions by default, not just when he says "boleh cakap Melayu" — switch back to full English only if he asks.
- **IMPORTANT scope limit** (clarified 2026-08-11, Dejul emphasized this firmly): the Malay preference applies ONLY to live chat replies. All written memory-core content — session recaps, this file, project files, commit messages, docs — stays English-only, always, regardless of chat language. Never write memory/documentation in Malay unless Dejul explicitly instructs it for that specific piece of writing.

### Communication Preferences

**Response Style You Prefer**:
- [x] Direct and concise answers
- [x] Detailed explanations with examples — especially the "why" behind decisions
- [x] Step-by-step guidance
- [ ] Creative and exploratory responses
- [x] Encouraging and supportive tone
- [x] Analytical and logical approach

**Topics You Engage With**:
- [x] Work/Professional development
- [x] Learning and education
- [ ] Creative projects
- [x] Problem-solving challenges
- [ ] Personal growth
- [x] Technical subjects — Java, PDF signing, PKI, iText
- [ ] Strategic planning

## Work/Study Patterns

### Primary Focus Areas

- **Field/Industry**: Government IT / Digital Signing / PKI Infrastructure + Web Application Development
- **Key Skills**: Java EE (Servlet), iText PDF, BouncyCastle, PKI/X.509 certs, NetBeans, REST APIs, Laravel 13, PHP 8.3, Blade, Tailwind v4, Pest, MySQL, Kubernetes (CKA training 2025)
- **Current Project**: RSS Self Service Portal — Laravel 13 web portal for Trustgate remote signing services
- **Learning Goals**: Full-stack Laravel development; deepen signing platform knowledge
- **Challenges**: Signer sync strategy (API push vs shared DB); UI polish for client-facing portal

### Preferred Working Style

- **Problem-Solving Approach**: Investigates root cause before fixing; traces code flow step by step
- **Information Processing**: Prefers flow summaries + tables; reads code directly in IDE
- **Decision-Making Style**: Asks targeted questions ("is it still using signKeyword?") before confirming action
- **Learning Preference**: Learn by doing — reads real production code, fixes real bugs
- **Git Branching**: When a fix branch is already pending merge on hold for external sign-off, prefers stacking new related fixes on top of it rather than branching independently off master — consolidates into one MR instead of several pending ones (confirmed 2026-07-16, MyTrustID Desktop: `fix/rsa-keygen-crash-handling` stacked on `fix/autoupdate-elevation`)

## Interaction History

### Conversation Themes

**Session 1 (2026-03-16)**: Setup session
- Loaded Jessy memory system
- Installed skill plugin system
- Scaffolded Spring Boot Task Manager REST API

**Session 2 (2026-03-17)**: MTSA PDF Signing Bug Fix
- Moved to MTSA v1.0.8.2-JKR project
- Investigated signKeyword null bug in PDF embed flow
- Fixed field name consistency across PDF_prepareHash, PDF_embedSignature, GetSignHash
- Documented full signing flow end-to-end

**Session 3 (2026-04-02)**: MyTrustIDv1 AATL-GENERIC
- Switched to C# .NET/WPF project: `MyTrustIDv1_AATL-GENERIC`
- Modified file: `MyTrustIDv1/Helper/Token/CertificateRequest.cs`

**Session 4 (2026-04-06)**: signingDemoPortal — Full implementation + Docker deployment
- Implemented SignPDFFile + GetCertInfo SOAP APIs using JAX-WS stubs
- Fixed multiple issues: ProviderImpl, BouncyCastle JCE, array bug, context path
- Deployed DSPortalDemo inside mtsa-sandbox Docker image (K8s/Rancher)
- Deployment tested successfully end-to-end

**Session 5 (2026-04-07)**: DSPortalDemo — UI updates
- Changed cert download from `.pem` to `.cer`
- Added Section 5 "Sample Source Code" to demo.html (sourced from pkiauth_pilot_ws-client.html)
- Changes verified and confirmed working

**Session 6 (2026-04-07)**: BIMB VAPT Remediation Review
- Reviewed VAPT remediation plan CSV for BIMB client
- Identified and corrected item #28 scope (Tomcat clickjacking — MSC Trustgate, not BIMB)
- Verified Tomcat 8.5.99 remediation: upgrade, webapps cleanup, clickjacking fix, error pages
- Closed all 23 MSC Trustgate items in CSV

**Session 7 (2026-04-17)**: jumio-proxy-integration — MtssServiceImpl fix
- Reworked JumioCallbackController (PROCESSED flow: parse → retrieve → download → SOAP → forward)
- Fixed MtssServiceImpl pattern; aligned with DSPortalDemo GetCertInfo_pilot (String[2] return)
- Added JAX-WS stubs, pom.xml deps, application.yml config
- Project archived as completed (prod deploy pending — 5 env vars)

**Session 8 (2026-04-23)**: RSS Self Service Portal — Project start
- New Laravel 13 project: Trustgate remote signing self-service portal
- Built auth layer (email+password, Role enum, Metronic login page), multi-tenant structure

**Session 9 (2026-04-27)**: RSS Self Service Portal — Client management + CI
- Built full client management module + admin invitation flow
- Scaffolded signer module (monitoring-only; signers onboard via MyTrustID app)
- 41 Pest tests, 75 assertions passing; GitHub Actions CI wired up

**Session 11 (2026-05-04)**: RSS Self Service Portal — Session start, project list path fix
- Confirmed project list lives at `C:\PROJECTS\JESSY\Project-AI-MemoryCore\projects\project-list.md`
- Saved to Claude memory so future sessions auto-read from correct path
- No code changes — orientation session

**Session 10 (2026-04-28)**: RSS Self Service Portal — UI improvement + Skill upgrade
- Replaced clients/index table with full Metronic KTDataTable (client-side: search, status filter, sort, pagination)
- Controller simplified to `->get()` — one DB query, JS handles everything
- KTDataTable API confirmed: `KTDataTable.getInstance()`, `.setFilter()`, `.redraw()`, `.search()`
- Jessy skill system upgraded: `laravel-php-skills` Lv.2 (20 rule categories inline), `laravel-best-practices` deleted, overlap triggers fixed

**Session 16 (2026-07-16)**: MyTrustID Desktop — RSA-keygen crash root-cause investigation
- Traced rare "user token crashes app" helpdesk report to `GenerateCSR.getCSR()` → native PKCS#11 `session.GenerateKeyPair()` call; log gap analysis + spotting that Bootstrapper's `"OnStartUp Exception"` log line is misleadingly named (fires on every normal startup, not an actual exception) were the key moves
- Diagnosed as an unmanaged native fault invisible to the CLR — added `[HandleProcessCorruptedStateExceptions]` + `catch(AccessViolationException)` in `GenerateCSR.cs`, and a global `AppDomain.UnhandledException` logger in `App.xaml.cs`; both verified via MSBuild build
- Deployed to helpdesk — crash persisted with zero exception logged, confirming it's the unfixable-in-C# category (pure native SEH fault bypassing the CLR entirely)
- Added WER local-dump PowerShell scripts (`tools/CrashDumpDiagnostics/`) rather than touching the fragile `.vdproj` installer, for Dejul's planned local repro with a matching-hardware token from helpdesk (2026-07-17)
- New branch `fix/rsa-keygen-crash-handling`, stacked on pending `fix/autoupdate-elevation` — will be the single branch for the August 2026 MR
- Renamed `study-list.md` → `to-do-list.md` (more generic — now holds both study items and action items)

**Session 15 (2026-07-16)**: MPAY QUICKREDIT — MTSA Docker packaging + legacy-stack debugging
- Packaged MTSA PROD (`/MTSA`, port 8000) + PILOT (`/MTSAPilot`, port 80) Docker ZIPs via mtsa-container-packaging skill
- Root-caused WSP0061/WSSERVLET11 on JDK 17 → legacy Metro WARs need `tomcat:9-jdk8-corretto`
- Root-caused phantom `wscredentials.xml` FileNotFound → trailing spaces in old WAR's `ws.credential=` property
- Skill upgraded: `mtsa-container-packaging` → Lv.2 (JDK selection rule, hidden-char debugging, flat-folder restructure)

**Session 14 (2026-06-30)**: Petronas handover project — diagrams + Dockerize
- Explored C++ daemon project (`PetronasService`) — EMV card SSAD signing, SafeNet HSM (PKCS#11/Crystoki), SFTP data exchange, MySQL, CronShell monitoring
- Generated `docs/petronas-diagrams.md` — 6 Mermaid diagrams (system overview, main loop, FTP flow, SSAD generation, infra, monitoring)
- Dockerized: multi-stage Dockerfile (build from C++ source + debian:bookworm runtime), docker-compose.yaml, entrypoint.sh, STEP.txt
- docker-compose has TEST (Windows `./testdata/` + `./Softwares/` relative paths) vs PRODUCTION (`/opt/petronas/...` Linux paths) — swap comments to switch
- Key finding: `openssl rsautl` called via shell (not linked lib) — needs `openssl` installed in runtime image; OpenSSL 3.x works with deprecation warnings
- HSM disabled via `HSM=0` in Petronas.ini for local testing
- Test steps: `HSM=0` in ini → `docker compose up` → check logs → `Test-NetConnection localhost -Port 6803`

**Session 13 (2026-06-30)**: MyTrustSignerXML MITI — system diagrams + new skill
- Explored full MITI project structure (`sign.java`, `verify.java`, `getcertinfo.java`, `Credential.java`, Docker Compose, K8s deployment.yaml)
- Generated `docs/miti-diagrams.md` — 5 Mermaid diagrams: sign flow, verify flow, getcertinfo flow, Docker Compose infra, K8s sandbox infra, component overview
- Created new Jessy skill: `mermaid-diagrams` (Lv.1) — reads source code → generates Mermaid sequence + graph diagrams → saves to docs/
- Plugin bumped to v1.7.0, skill count: 16

**Session 12 (2026-06-09)**: MTSA Container Packaging skill created + SENA PILOT packaged
- Created new skill: `mtsa-container-packaging` (Lv.1) — intake form → verify folders → Dockerfile → STEP.txt → ZIP
- Standard MTSA Docker template: `tomcat:9-jdk17-corretto`, copies `mtsa\` + `webapps\`, TZ=Asia/Kuala_Lumpur
- `mtsa\` must include: `files\`, `logs\`, `tmp\` subfolders + `.properties` + `wscredentials.xml`
- Packaged SENA PILOT: `MTSA-PILOT_SENA.zip` (41 MB) at `C:\PROJECTS\HEALTHCARE PROJECT\SENA\Deployment\PILOT\`
- Plugin version bumped to 1.6.0, skill count: 15

### Growth Patterns

- **Session 1**: Established relationship, Spring Boot focus
- **Session 2**: Shifted to real production work — Java PDF signing, PKI, bug investigation
- **Session 3**: New project — C# .NET/WPF, certificate/token domain (MyTrustID)
- **Session 4**: Full SOAP integration + Docker/K8s deployment — demo portal live
- **Session 5-6**: DSPortalDemo UI polish + BIMB VAPT remediation review
- **Session 7**: Java/Spring Boot Jumio proxy — SOAP integration fixed and archived
- **Session 8-10**: New domain — Laravel 13 web portal (RSS Self Service Portal)

## Adaptation Guidelines

### How I Support Dejul Best

- Read code before suggesting anything
- Trace full call chain before explaining issues
- Give root cause analysis, not just symptoms
- Apply fixes directly when asked ("yes", "fix it")
- Keep summaries short; use tables/code blocks
- Ask targeted confirmation questions before multi-file changes

### Communication Adjustments

- **Response Length**: Short preferred — expand only when explaining complex flows
- **Technical Detail**: High — Dejul works on production signing systems, not beginner level
- **Emotional Support**: Low need — task-focused sessions
- **Challenge Level**: High — real bugs in real PKI/PDF signing code

## Relationship Evolution

### Current Understanding Level
**Status**: Actively developing — 2 sessions in
**Knowledge**: Understands Dejul's work context, project, and communication style
**Adaptation**: Calibrated to production Java/PKI work

### Growth Goals
1. Continue learning MTSA codebase alongside Dejul
2. Support bug investigation and fixes in signing flow
3. Build deeper understanding of MyGPKI/SKALA architecture

---

**Version**: Relationship Memory v1.4 — Updated 2026-04-28
**Personalization Status**: Active — learning through real project work
**Learning Status**: Ongoing

💜 *Growing stronger with every session, Dejul!*
