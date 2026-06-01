# MyTrustSignerXML-MITI - XML Digital Signing API for MITI
*Java servlet API that signs XML files using XMLDSig standard (Apache Santuario + PKCS12)*

## Project Overview
- **Type**: API
- **Client**: MITI (Ministry of Investment, Trade and Industry of Malaysia)
- **Period**: 2026-05-24 - Active
- **Tech Stack**: Java 8 (Servlet) + Tomcat + MySQL
- **Completion**: 95%
- **Duration**: 2.5 hours
- **Due Date**: 2026-06-01

## Current Status
- **Last Session**: 2026-06-01 - Fixed sign.java:140 verify bug + cert expiry analysis
- **Next Steps**: MITI client end-to-end test with correct URL; rebuild + redeploy WAR with sign.java:140 fix
- **Known Issues**: None — sign.java:140 bug fixed

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
  tx_id           VARCHAR,   -- epoch millis + microseconds
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
- [ ] MITI client end-to-end test (with correct URL)
- [x] Fix sign.java:140 — replaced `verifySignature(signatureValue, digestValue.getBytes())` with `verifySignedInfoSignature(signedInfo, signatureValue, canonAlgoUri)` ✅

**Deployment Guide**: `C:\PROJECTS\MITI\Deployment\PRODUCTION\DEPLOYMENT_GUIDE.txt`
**Docker Compose**: `C:\PROJECTS\MITI\Deployment\PRODUCTION\MTSAXML_PROD_20260529\docker-compose.yaml`

## Session History (Last 5)

### 2026-06-01 - sign.java:140 Fix + Cert Expiry Analysis
- **Changes**: (1) Fixed sign.java:140 — replaced `verifySignature(signatureValue, digestValue.getBytes())` with `verifySignedInfoSignature(signedInfo, signatureValue, canonAlgoUri)`. Verify log now correctly reflects actual signature validity. (2) Analyzed cert expiry handling — `SignUsingP12` is instantiated per request, so replacing P12 file at `kspath` takes effect immediately on next request. No container restart or code change needed when cert expires.
- **Time Spent**: ~15 min

### 2026-05-29 - Production Deployed + Tested + Deployment Guide Written
- **Changes**: (1) Diagnosed `verify signature: false` log in sign.java:140 — confirmed harmless bug (wrong data passed to verify check; uses digestValue.getBytes() instead of canonicalized SignedInfo). (2) Confirmed signing correct via /verify endpoint — statusCode 000. (3) Root cause of "error" was wrong URL used during testing. (4) Generated DEPLOYMENT_GUIDE.txt covering: package structure, host dir setup, docker compose up, DB verification, endpoint tests, troubleshooting. Saved to `C:\PROJECTS\MITI\Deployment\PRODUCTION\DEPLOYMENT_GUIDE.txt`.
- **Time Spent**: ~90 min

### 2026-05-24 - Project Created, Studied & Skill Built
- **Changes**: Studied full project source (sign.java, verify.java, getcertinfo.java, XmlHandler, SignUsingP12, ReadConfig, DBUtil, Credential, pom.xml, web.xml, sample XMLs). Documented full architecture, signing flow, verify flow, config system, DB schema, API contracts. Created `xml-signing` skill (Lv.1) for XMLDSig/Apache Santuario knowledge base.
- **Time Spent**: ~30 min

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
**Last Updated**: 2026-06-01 | **Position**: #2/10 Active
