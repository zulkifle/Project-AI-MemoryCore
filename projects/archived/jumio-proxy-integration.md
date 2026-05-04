# jumio-proxy-integration - Adacash–Trustgate Jumio Proxy Service
*Spring Boot proxy layer between Adacash and Jumio for eKYC integration*

## Project Overview
- **Type**: Web Service / Proxy API (Spring Boot)
- **Period**: 2026-04-16 - 2026-04-22
- **Tech Stack**: Java 17 + Spring Boot 3.0.9 + OpenFeign + Maven + Docker + Kubernetes
- **Completion**: 100%
- **Duration**: ~3 sessions
- **Status**: Completed — dev done, pending production deployment (5 env vars to update manually)

## Project Context
- **Client**: Adacash Sdn Bhd
- **Our Company**: Trustgate Sdn Bhd
- **Integration**: Trustgate acts as proxy between Adacash and Jumio eKYC API

## Flow Summary
1. Adacash routes eKYC requests through Trustgate proxy
2. Proxy returns Jumio landing page URL to user
3. User completes eKYC on Jumio side
4. Jumio sends callback to Trustgate callback URL
5. On `PROCESSED` status: retrieve workflow, download images as Base64, call MTSS SOAP, forward to Adacash
6. On non-`PROCESSED` status: log and return 200

## Current Status
- **Last Session**: 2026-04-22 - Production deployment-prod.yaml reviewed, pending prod env values from client
- **Next Steps**: Update 5 prod values in `deployment-prod.yaml`, then `kubectl apply -f deployment-prod.yaml`
- **Known Issues**: None

## Pending Production Values (deployment-prod.yaml)
| Env Var | Status |
|---|---|
| `image` tag | Update from `jumioproxy:1.0` to latest (pilot is `1.06`) |
| `JUMIO_CALLBACK_BASE_URL` | Remove `_pilot` from path |
| `ADACASH_CALLBACK_URL` | Replace `preprod.getcredit.my` with production URL |
| `MTSS_USERNAME` | Replace `rockWing_pilot` with production username |
| `MTSS_PASSWORD` | Replace with production password |

## Session History (Last 5)

### 2026-04-22 - Production Review & Archive
- **Changes**: Reviewed deployment-prod.yaml vs pilot yaml, identified 5 env vars needing production values. Same .jar/image confirmed reusable across environments.
- **Time Spent**: ~15 min

### 2026-04-21 - Image Compression Implementation
- **Changes**: Added ImageCompressionUtil + Scalr, wired into callback controller flow before MTSS SOAP call.
- **Time Spent**: ~1 session

### 2026-04-17 - Implementation Complete
- **Changes**: Full Trustgate integration — JumioCallbackController reworked, MTSS SOAP wired, images download, JAX-WS stubs generated.
- **Note**: Session was lost (not saved); changes recovered from git diff
- **Time Spent**: ~1 session

### 2026-04-16 - Project Created
- **Changes**: Initial project setup, reviewed PROM.txt and pom.xml
- **Time Spent**: ~0 min

## Historical Summary
Project started 2026-04-16 as Trustgate proxy layer between Adacash and Jumio eKYC API. Key milestones: full callback flow implemented (MTSS SOAP, image download, Adacash forward), image compression added before SOAP call. Pilot deployed and tested. Production deployment pending 5 env var values from client — same .jar, same image, just different config.

## Technical Notes
- **Repository**: `C:\PROJECTS\DOCKER GITLAB\docker\jumio-proxy\app\jumio-proxy\`
- **Artifact ID**: `adacash-trustgate-jumio-proxy` v0.0.1
- **Pilot yaml**: `pilot/deployment.yaml` — image `jumioproxy-pilot:1.06`, nodePort 30241
- **Prod yaml**: `production/deployment-prod.yaml` — image `jumioproxy:1.0`, nodePort 30240
- **Deploy command**: `kubectl apply -f deployment-prod.yaml`
- **Key Dependencies**: Spring Boot 3.0.9, OpenFeign 4.0.3, Lombok, Commons Lang3
- **SOAP Integration**: MTSS CallbackJumio API (downstream)
- **Auth**: Jumio Basic Auth → OAuth 2.0 Bearer token flow

---
**Last Updated**: 2026-04-22 | **Status**: Completed — Archived (prod deploy pending)
