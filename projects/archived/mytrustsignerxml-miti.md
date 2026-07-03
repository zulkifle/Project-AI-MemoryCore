# MyTrustSignerXML-MITI - XML Digital Signing API for MITI
*Java servlet API that signs XML files using XMLDSig standard (Apache Santuario + PKCS12)*

## Project Overview
- **Type**: API
- **Client**: MITI (Ministry of Investment, Trade and Industry of Malaysia)
- **Period**: 2026-05-24 - 2026-06-18
- **Tech Stack**: Java 8 (Servlet) + Tomcat + MySQL
- **Completion**: 100%
- **Duration**: 3 hours
- **Due Date**: 2026-06-01

## Current Status
- **Last Session**: 2026-06-18 - Project closed. MITI confirmed pilot OK, PROD package handed over, MITI self-deploys.
- **Next Steps**: None — fully handed over. MITI owns deployment.
- **Known Issues**: None. All bugs fixed and compiled into PROD package.

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
- [x] PROD package handed over to MITI ✅ — MITI self-deploys (2026-06-18)

**Deployment Guide**: `C:\PROJECTS\MITI\Deployment\PRODUCTION\DEPLOYMENT_GUIDE.txt`
**Docker Compose**: `C:\PROJECTS\MITI\Deployment\PRODUCTION\MTSAXML_PROD_20260529\docker-compose.yaml`

## Session History (Last 5)

### 2026-06-18 - Chain Bug Traced + Project Closed
- **Changes**: MITI confirmed pilot OK. Reviewed unrecorded chain fix (June 8 session): `SignUsingP12` changed from `getCertificate(alias)` → `getCertificateChain(alias)[0]` — production P12 has a full cert chain embedded; old code could return wrong cert causing "signature invalid". Fix was compiled into PROD package (June 11). PROD package handed over to MITI — they self-deploy. Project fully closed on our side.
- **Time Spent**: ~15 min

### 2026-06-11 - Fixed Duplicate tx_id Race Condition
- **Trigger**: PILOT log (2026-06-05) showed `SQLIntegrityConstraintViolationException: Duplicate entry '1780630985393827669' for key 'tx_id'`. Dejul asked if it was fixed / would recur.
- **Root cause (two independent paths)**:
  1. `DBUtil.getTXID()` built the id from `currentTimeMillis + (nanoTime/1000)%1e6`. Two threads reading `nanoTime` within ~1µs produce the same micro portion → same id under concurrency.
  2. `sign.java` held `txid` and `db` as **servlet instance fields**. Servlets are singletons, so concurrent requests shared them — request B's `getTXID()` overwrote request A's `txid` before A saved → both saved the same id. Plus catch blocks re-saved the same `txid` (third path).
- **Fix**: `DBUtil.java` uses `AtomicLong` seq; `sign.java` moved `txid`/`db` to request-local vars with `txSaved` guard.
- **Time Spent**: ~45 min

### 2026-06-11 - PILOT Billing Verified — PROD Blocked on Client
- **Changes**: Confirmed steps 1-4 complete — WAR built, billing.class deployed to PILOT, `/billing` endpoint tested OK, recipients restored to all 4.
- **Time Spent**: ~10 min

### 2026-06-05 - Monthly Billing Feature (billing.java + cron)
- **Changes**: `/billing` servlet — crond triggers 1st of month 08:00 MYT, queries successful tx for previous month, generates CSV, emails to recipients via SMTP.
- **Time Spent**: ~2.5 hours

### 2026-06-01 - sign.java:140 Fix + Cert Expiry Analysis
- **Changes**: Fixed sign.java:140 verify bug. Analyzed cert expiry — replacing P12 at `kspath` takes effect on next request, no container restart needed.
- **Time Spent**: ~15 min

## Historical Summary
Project started 2026-05-24: studied full source, documented architecture (sign/verify/getcertinfo flows, config, DB schema, API contracts), and created the `xml-signing` skill (Lv.1) for XMLDSig/Apache Santuario. Reached 100% functional completion by end of May — production deployed & tested, DEPLOYMENT_GUIDE written. Key milestones since: sign.java:140 verify fix, monthly `/billing` servlet (cron-driven CSV email), PILOT billing verified, and the duplicate-tx_id race fix.

## Technical Notes
- **Repository**: C:\PROJECTS\MITI\Development\MyTrustSignerXML
- **Key Dependencies**: Apache Santuario xmlsec 4.0.1, PKCS12 keystore, MySQL
- **WAR Context Path**: `/SandboxAPI2/MyTrustSignerXMLPilot` (sandbox)
- **Java Version**: Java 8 (compiled target 1.8)
- **XMLDSig Standard**: W3C XML-Signature Syntax and Processing
- **Canonicalization**: EXC-C14N — exclusive C14N, omit comments
- **Signature Algorithm**: RSA-SHA256
- **Digest Algorithm**: SHA-256

---
**Last Updated**: 2026-07-03 | **Status**: Archived (LRU) — Completed ✅ 2026-06-18. PROD handed over to MITI.
