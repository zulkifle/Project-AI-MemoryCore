# MyTrustSignerXML-MITI - XML Digital Signing API for MITI
*Java servlet API that signs XML files using XMLDSig standard (Apache Santuario + PKCS12)*

## Project Overview
- **Type**: API
- **Client**: MITI (Ministry of Investment, Trade and Industry of Malaysia)
- **Period**: 2026-05-24 - Active (reactivated 2026-06-05)
- **Tech Stack**: Java 8 (Servlet) + Tomcat + MySQL
- **Completion**: 100%
- **Duration**: 2.5 hours
- **Due Date**: 2026-06-01

## Current Status
- **Last Session**: 2026-06-11 - Fixed duplicate `tx_id` race condition (DBUtil + sign.java), code-level only
- **Next Steps**: PROD package prepared & verified (tx_id fix compiled in, DB_URL→mysql, v1.1, UKM artifacts purged). Dejul to zip + deploy. ⏸️ Go-live gated on MITI pilot sign-off. On deploy: update `DB_URL` on host `/opt/mtsa/properties/mtsa.properties` + `docker compose build --no-cache`
- **Known Issues**: None open. Duplicate `tx_id` (SQLIntegrityConstraintViolationException, seen in PILOT logs 2026-06-05) fixed in dev source — pending WAR rebuild + redeploy to take effect

## Architecture

### Endpoints (web.xml)
| Endpoint | Method | Function |
|----------|--------|----------|
| `/sign` | POST | Sign XML file (base64 in → base64 signed XML out) |
| `/verify` | POST | Verify signed XML (hash + signature check) |
| `/getcertinfo` | GET | Return P12 cert details (serial, subject, validity) |

### Signing Flow (`sign.java`)
1. `Init.init()` — Apache Santuario init
2. `ReadConfig` — loads `config.properties` → external `.properties` file (pilot or prod)
3. `Credential.AuthenticateHdr` — Basic Auth via `credential.json`
4. Parse JSON body: `{ "base64xml": "..." }`
5. `XmlHandler` — parse XML, extract `<Object Id="ObjectContent">` element
6. `XmlDigest.getDocDigest(objectEl)` — EXC-C14N canonicalize → SHA-256 → base64 DigestValue
7. `XmlHandler.updateXMLElement(digestValue)` — set algorithms + DigestValue in DOM
8. `SignUsingP12.signSignedInfo(signedInfo, canonAlgo)` — canonicalize `<SignedInfo>` → RSA-SHA256 sign → base64
9. `XmlHandler.updateXMLValue(...)` — embed `<SignatureValue>` into final XML
10. Save signed file + `db.saveTxDb(txid, digest, signature, status)`
11. Return JSON: `{ statusCode, txID, signedData (base64), certX509, subjectDN }`

### XML Structure (XMLDSig standard)
```xml
<Signature xmlns="http://www.w3.org/2000/09/xmldsig#">
  <SignedInfo>
    <CanonicalizationMethod Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
    <SignatureMethod Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256"/>
    <Reference URI="#ObjectContent">
      <DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>
      <DigestValue>base64...</DigestValue>
    </Reference>
  </SignedInfo>
  <SignatureValue>base64...</SignatureValue>
  <KeyInfo><KeyName>malpcoeodes</KeyName></KeyInfo>
  <Object Id="ObjectContent">
    <Package>...</Package>   <!-- MITI payload XML -->
  </Object>
</Signature>
```

### Verification Flow (`verify.java`)
1. Auth check (same as sign)
2. Parse JSON: `{ "signedXmlBase64": "..." }`
3. `XmlElementExtractor` — XPath extract DigestValue + SignatureValue from signed XML
4. `verifyHash` — re-canonicalize `<Object>`, recompute SHA-256, compare to DigestValue
5. `verifySignedInfoSignature` — re-canonicalize `<SignedInfo>`, verify RSA-SHA256 against P12 public key

### Config System
```
WEB-INF/classes/config.properties
  → app.config = /opt/miti/   (base path)
  → app.stage  = pilot | production

/opt/miti/properties/mtsa-pilot.properties     (pilot)
/opt/miti/properties/mtsa.properties           (production)
  → kspath     = path to .p12 file
  → kspass     = P12 password
  → DB_URL/DB_USER/DB_PASSWORD
  → jsonAuth   = credential.json filename
  → service    = 200 (OK) | 503 (maintenance)
  → debug      = true|false
```

### Auth System
- Basic Auth header decoded → username:password
- Validated against JSON file: `[{ "username": "...", "password": "..." }]`
- Path: `{app.config}/properties/{jsonAuth}`

### Database (MySQL)
```sql
transactions (
  tx_id           VARCHAR,   -- NOT NULL UNIQUE; epoch millis + 6-digit AtomicLong seq
  digest_value    VARCHAR,
  signature_value VARCHAR,
  status          VARCHAR,   -- "Success" | "Failed" | error msg
  tx_timestamp    DATETIME
)
```

## Key Files
```
src/main/java/com/msctg/miti/mytrustsigner/
├── sign.java                    — /sign servlet (main signing endpoint)
├── verify.java                  — /verify servlet
├── getcertinfo.java             — /getcertinfo servlet
├── SignRes.java                 — Sign response DTO
├── VerifyRes.java               — Verify response DTO
├── certRes.java                 — Cert info response DTO
├── xmlhandler/
│   ├── XmlHandler.java          — Core XML DOM manipulation + canonicalization
│   ├── SignUsingP12.java        — PKCS12 load, sign, verify (Apache Santuario)
│   ├── ReadConfig.java          — Config loader (2-tier: classpath + external props)
│   ├── XmlDigest.java           — Digest computation for <Object> element
│   ├── Signature.java           — DTO for extracted XMLDSig elements
│   └── XmlFileReader.java       — Base64 XML decode + file hash util
├── authentication/
│   ├── Credential.java          — Basic Auth handler (JSON credential file)
│   └── AuthHeader.java          — Auth header model
└── util/
    ├── DBUtil.java              — MySQL tx save + TXID generator
    ├── DatabaseConfig.java      — JDBC connection factory
    ├── CanonicalizationAlgo.java — C14N algorithm enum (default: EXC-C14N)
    ├── ConvertUtils.java        — Base64 encode/decode, file save
    ├── Debugging.java           — Debug-mode logger (controlled by config)
    ├── KeyNameUtil.java         — KeyName random generator
    └── X509Handler.java         — DN transformation util

webapps/SandboxAPI2#MyTrustSignerXMLPilot/  — deployed sandbox WAR exploded
  WEB-INF/web.xml                           — servlet mappings

pom.xml
  → org.apache.santuario:xmlsec:4.0.1
  → com.mysql:mysql-connector-j:8.2.0
  → log4j 2.18.0, gson 2.8.9, org.json
  → JAX-WS (PKIWS v2 WSDL — pkiws stubs)

TestSignXML/
  SANDBOX-API-ORINGINAL.xml   — sample unsigned XML
  SANDBOX-API-SIGNED.xml      — sample signed XML (for reference/testing)
```

## Production Deployment Checklist
- [x] Create `/opt/mtsa/properties/mtsa.properties` with prod `kspath`, `kspass`, DB creds
- [x] Place production `.p12` signing cert at `kspath`
- [x] Create `credential.json` with authorized client credentials
- [x] Create MySQL `transactions` table (auto via init.sql + docker compose)
- [x] Update `config.properties` → `app.stage=production`
- [x] Build WAR + deploy via Docker (`docker compose up -d`)
- [x] Test `/getcertinfo` ✅
- [x] Test `/sign` ✅ — statusCode 000, signed XML produced
- [x] Test `/verify` ✅ — signed XML verified successfully
- [x] MITI client end-to-end test (with correct URL) ✅
- [x] Fix sign.java:140 — replaced `verifySignature(signatureValue, digestValue.getBytes())` with `verifySignedInfoSignature(signedInfo, signatureValue, canonAlgoUri)` ✅
- [x] Add `/billing` servlet — monthly CSV email via crond (production only) ✅ (code done)
- [x] Build WAR + copy to PILOT webapps → test `/billing` endpoint → restore recipients to all 4 ✅
- [ ] Copy built WAR to PROD deployment folder → run DEPLOYMENT_CHECKLIST.txt → redeploy ⏸️ (awaiting MITI pilot sign-off)

**Deployment Guide**: `C:\PROJECTS\MITI\Deployment\PRODUCTION\DEPLOYMENT_GUIDE.txt`
**Docker Compose**: `C:\PROJECTS\MITI\Deployment\PRODUCTION\MTSAXML_PROD_20260529\docker-compose.yaml`

## Session History (Last 5)

### 2026-06-11 - Fixed Duplicate tx_id Race Condition
- **Trigger**: PILOT log (2026-06-05) showed `SQLIntegrityConstraintViolationException: Duplicate entry '1780630985393827669' for key 'tx_id'`. Dejul asked if it was fixed / would recur.
- **Root cause (two independent paths)**:
  1. `DBUtil.getTXID()` built the id from `currentTimeMillis + (nanoTime/1000)%1e6`. Two threads reading `nanoTime` within ~1µs produce the same micro portion → same id under concurrency.
  2. `sign.java` held `txid` and `db` as **servlet instance fields**. Servlets are singletons, so concurrent requests shared them — request B's `getTXID()` overwrote request A's `txid` before A saved → both saved the same id. Plus catch blocks re-saved the same `txid` (third path).
- **Fix (code-level only, NO DB change — `UNIQUE` constraint kept as backstop)**:
  - `DBUtil.java`: `getTXID()` now uses a process-wide `AtomicLong` sequence (`currentTimeMillis + %06d seq`) — cannot collide in-process. Added dedicated `SQLIntegrityConstraintViolationException` catch for clear logging.
  - `sign.java`: moved `txid`/`db` to request-local vars; added `txSaved` guard so catch blocks never re-insert; null-guarded catch saves (also fixes latent NPE when exception fires before `db` created).
- **Verified**: `mvn compile` → EXIT=0.
- **PROD package prep** (`Deployment\PRODUCTION\MTSAXML_PROD`): Dejul rebuilt WAR (DBUtil/sign .class stamped today, confirmed `AtomicLong` + constraint-catch compiled in). Reviewed package vs `DEPLOYMENT_CHECKLIST.txt`:
  - Fixed `mtsa.properties` `DB_URL` `localhost` → `mysql` (compose service name — would've failed DB connect inside container).
  - Bumped image tag `mtsa-miti-prod:1.` → `1.1` in docker-compose.yaml.
  - Removed leaked UKM artifacts from webapps (`SandboxAPI#MTSAPilotUKM.war` + 2 `.filepart` leftovers, ~75 MB).
  - Left default Tomcat apps (docs/examples/manager/host-manager/ROOT) in place per Dejul's call.
- **Deploy caveat**: compose mounts `/opt/mtsa/properties` from host → host's `mtsa.properties` overrides baked-in copy; must update `DB_URL` on the server host too. Rebuild with `docker compose build --no-cache`.
- **Pending**: Dejul to zip package + deploy. PROD go-live still gated on MITI pilot sign-off.
- **Time Spent**: ~45 min

### 2026-06-11 - PILOT Billing Verified — PROD Blocked on Client
- **Changes**: Confirmed steps 1-4 complete — WAR built, billing.class deployed to PILOT, `/billing` endpoint tested OK, recipients restored to all 4. Project now effectively done on our side.
- **Status**: PROD deploy is the only remaining task, gated on MITI confirming the pilot test is successful. No action on our side until client responds.
- **Time Spent**: ~10 min

### 2026-06-05 - Monthly Billing Feature (billing.java + cron)
- **New Feature**: `/billing` servlet — crond triggers on 1st of every month at 08:00 MYT, queries successful transactions for previous month, generates CSV, emails to recipients via SMTP (Gmail app password)
- **Dev changes** (`C:\PROJECTS\MITI\Development\MyTrustSignerXML`):
  - `Dockerfile` — added `cron` + `curl` install, `COPY crontab`, `COPY entrypoint.sh`, replaced `CMD` with `ENTRYPOINT ["/entrypoint.sh"]`
  - `crontab` — new file: `0 8 1 * *` curl to `/billing`
  - `entrypoint.sh` — new file: `service cron start` then `catalina.sh run`
  - `mtsa-pilot.properties` — DB_URL changed from `localhost` to `mysql` (compose service name); added `billing.recipients`, `smtp.*`; testing: recipients set to `zulkifle@msctrustgate.com` only
  - `billing.java` — (1) production-only guard: returns `{"status":"SKIPPED"}` if `stage != production`; (2) email subject hardcoded to `MITI PRODUCTION Monthly Billing Report — {monthLabel}`
- **PILOT deployment changes** (`C:\PROJECTS\MITI\Deployment\PILOT\MTSAXML_PILOT`):
  - `mtsa-pilot.properties` — fixed DB_URL + added SMTP/billing config
  - `docker-compose.yaml` — added `networks: default: bridge`
- **PROD deployment changes** (`C:\PROJECTS\MITI\Deployment\PRODUCTION\MTSAXML_PROD`):
  - `mtsa.properties` — fixed DB_URL (`mysql`), added SMTP/billing config
  - `crontab` — removed duplicate pilot line
  - `docker-compose.yaml` — added `networks: default: bridge`
  - `DEPLOYMENT_CHECKLIST.txt` — new file: 9-section pre-deployment checklist (reusable for other projects)
- **Pending**: WAR not yet rebuilt/copied to either deployment folder (billing.class missing)
- **Time Spent**: ~2.5 hours

### 2026-06-01 - sign.java:140 Fix + Cert Expiry Analysis
- **Changes**: (1) Fixed sign.java:140 — replaced `verifySignature(signatureValue, digestValue.getBytes())` with `verifySignedInfoSignature(signedInfo, signatureValue, canonAlgoUri)`. Verify log now correctly reflects actual signature validity. (2) Analyzed cert expiry handling — `SignUsingP12` is instantiated per request, so replacing P12 file at `kspath` takes effect immediately on next request. No container restart or code change needed when cert expires.
- **Time Spent**: ~15 min

### 2026-05-29 - Production Deployed + Tested + Deployment Guide Written
- **Changes**: (1) Diagnosed `verify signature: false` log in sign.java:140 — confirmed harmless bug (wrong data passed to verify check; uses digestValue.getBytes() instead of canonicalized SignedInfo). (2) Confirmed signing correct via /verify endpoint — statusCode 000. (3) Root cause of "error" was wrong URL used during testing. (4) Generated DEPLOYMENT_GUIDE.txt covering: package structure, host dir setup, docker compose up, DB verification, endpoint tests, troubleshooting. Saved to `C:\PROJECTS\MITI\Deployment\PRODUCTION\DEPLOYMENT_GUIDE.txt`.
- **Time Spent**: ~90 min

## Historical Summary
Project started 2026-05-24: studied full source, documented architecture (sign/verify/getcertinfo flows, config, DB schema, API contracts), and created the `xml-signing` skill (Lv.1) for XMLDSig/Apache Santuario. Reached 100% functional completion by end of May — production deployed & tested, DEPLOYMENT_GUIDE written. Key milestones since: sign.java:140 verify fix, monthly `/billing` servlet (cron-driven CSV email), PILOT billing verified, and the duplicate-tx_id race fix.

## Technical Notes
- **Repository**: C:\PROJECTS\MITI\Development\MyTrustSignerXML
- **Key Dependencies**: Apache Santuario xmlsec 4.0.1, PKCS12 keystore, MySQL
- **WAR Context Path**: `/SandboxAPI2/MyTrustSignerXMLPilot` (sandbox)
- **Java Version**: Java 8 (compiled target 1.8)
- **XMLDSig Standard**: W3C XML-Signature Syntax and Processing (https://www.w3.org/TR/xmldsig-core/)
- **Canonicalization**: EXC-C14N (`http://www.w3.org/2001/10/xml-exc-c14n#`) — exclusive C14N, omit comments
- **Signature Algorithm**: RSA-SHA256 (auto-detected from P12 key type; also supports ECDSA)
- **Digest Algorithm**: SHA-256

---
**Last Updated**: 2026-06-11 | **Status**: tx_id fix in PROD package ✅ (DB_URL→mysql, v1.1, UKM purged) — Dejul to zip+deploy; PROD blocked on MITI pilot sign-off ⏸️
