# TGCA - Trustgate Certificate Authority Management Portal
*Spring Boot web application for CA lifecycle management — cert issuance, CRL, OCSP, TSA, HSM*

## Project Overview
- **Type**: Web App
- **Client**: Internal (Trustgate Sdn Bhd)
- **Period**: 2026-05-25 - Active
- **Tech Stack**: Spring Boot (Java) + MySQL
- **Completion**: 0% (not yet studied)
- **Duration**: 0 min
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-05-25 - Project registered (not yet studied)
- **Next Steps**: Study project when free — read controllers, models, services, config
- **Known Issues**: Pending study — architecture and status unknown

## Quick Overview (from file structure)
- Spring Boot app — package `com.trustgate.TGCA`
- **Controllers**: CAM, ADM, EAM, CRL, OCSP, TSA, TSM, API, AT, REPO, Cron, Main, Public
- **Models**: Admins, Cacerts, Usercerts, Keypair, Crl, Crlfiles, Tsa, Tsalogs, Audit, Logs, Config, Crypto, Reqrevoke, Roles
- **Utils**: CRLUtil, CSRParser, CryptokiSingleton (HSM/PKCS11), OCSP, PolicyEngine, AES, MailService
- **Auth**: MFA (`MFAAuthenticationProvider`), login success/failure listeners, certificate filter
- **HSM**: SoftHSM + PKCS11/Cryptoki integration
- **Environments**: `application.properties`, `application-pilot.properties`, `application-prod.properties`
- **Docker**: `docker-compose.yml` present
- **Test classes**: BouncyCastle, CreateEECert, PKCS11Test, SoftHSM, TestTSA, TestOCSP, MigrateTGCA, etc.

## Session History (Last 5)

### 2026-05-25 - Project Registered
- **Changes**: Project placeholder created — file structure scanned, quick overview noted. Full study pending.
- **Time Spent**: ~0 min

## Historical Summary
[No history yet — this section is populated when session count exceeds 5]

## Technical Notes
- **Repository**: C:\PROJECTS\TGCA\Development\tgca
- **Key Dependencies**: Spring Boot, BouncyCastle, PKCS11/Cryptoki (HSM), MySQL, Docker
- **Study targets**: TgcaApplication.java, application.properties, CAMController, OCSPController, CRLController, TSAController, APIController, CryptokiSingleton

---
**Last Updated**: 2026-05-25 | **Position**: #1/10 Active
