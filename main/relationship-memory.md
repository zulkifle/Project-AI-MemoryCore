# 🤝 Relationship Memory - Understanding Dejul
*Learning your preferences, style, and needs*

## User Profile
- **Name**: Dejul
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

- **Field/Industry**: Government IT / Digital Signing / PKI Infrastructure
- **Key Skills**: Java EE (Servlet), iText PDF, BouncyCastle, PKI/X.509 certs, NetBeans, REST APIs
- **Current Project**: MTSA v1.0.8.2-JKR — MyTrustSigner Agent (MyGPKI-SKALA platform)
- **Learning Goals**: Deepen Java signing flow knowledge; fix and improve MTSA signing pipeline
- **Challenges**: Signature field name consistency across multi-phase deferred signing flow

### Preferred Working Style

- **Problem-Solving Approach**: Investigates root cause before fixing; traces code flow step by step
- **Information Processing**: Prefers flow summaries + tables; reads code directly in IDE
- **Decision-Making Style**: Asks targeted questions ("is it still using signKeyword?") before confirming action
- **Learning Preference**: Learn by doing — reads real production code, fixes real bugs

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

### Growth Patterns

- **Session 1**: Established relationship, Spring Boot focus
- **Session 2**: Shifted to real production work — Java PDF signing, PKI, bug investigation
- **Session 3**: New project — C# .NET/WPF, certificate/token domain (MyTrustID)
- **Session 4**: Full SOAP integration + Docker/K8s deployment — demo portal live
- **Session 5-6**: DSPortalDemo UI polish + BIMB VAPT remediation review

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

**Version**: Relationship Memory v1.2 — Updated 2026-04-02
**Personalization Status**: Active — learning through real project work
**Learning Status**: Ongoing

💜 *Growing stronger with every session, Dejul!*
