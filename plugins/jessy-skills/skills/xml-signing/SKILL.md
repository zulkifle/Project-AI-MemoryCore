---
name: xml-signing
description: >
  Use this skill for ALL tasks involving XML digital signing (XMLDSig W3C standard),
  Apache Santuario (xmlsec), XML canonicalization (C14N/EXC-C14N), PKCS12 signing in Java,
  or the MyTrustSignerXML-MITI project. Triggers on: XMLDSig, Apache Santuario, xmlsec,
  canonicalization, EXC-C14N, DigestValue, SignedInfo, SignatureValue, xml-exc-c14n,
  "sign XML", "verify XML signature", "XML signing", mtsa.properties, kspath, kspass,
  credential.json (in XML signing context), or any task referencing the MyTrustSignerXML project.
  Do NOT trigger for PDF digital signing (covered by mygpki-signing or signing-labs skills).
---

# XML Digital Signing (XMLDSig) — Expert Skill
*Java Servlet + Apache Santuario implementation of W3C XMLDSig standard — MyTrustSignerXML-MITI*

## Activation

When this skill activates:
`🔏 XML Signing Skill loaded — XMLDSig + Apache Santuario flow ready.`

Then apply the knowledge below to assist with the task.

---

## XMLDSig Structure (W3C Standard)

The project uses **enveloping XML signature** — the payload is INSIDE the `<Object>` element:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Signature xmlns="http://www.w3.org/2000/09/xmldsig#">
  <SignedInfo>
    <CanonicalizationMethod Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
    <SignatureMethod Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256"/>
    <Reference URI="#ObjectContent">
      <DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>
      <DigestValue>base64-sha256-of-canonicalized-Object...</DigestValue>
    </Reference>
  </SignedInfo>
  <SignatureValue>base64-rsa-sha256-of-canonicalized-SignedInfo...</SignatureValue>
  <KeyInfo>
    <KeyName>malpcoeodes</KeyName>     <!-- MITI client identifier -->
  </KeyInfo>
  <Object Id="ObjectContent">
    <Package>
      ... MITI payload XML ...
    </Package>
  </Object>
</Signature>
```

**Key rule:** `<SignatureValue>` signs `<SignedInfo>`, NOT the payload directly.
The payload is protected via the digest chain: `Object → DigestValue → SignedInfo → SignatureValue`.

---

## Signing Flow (sign.java)

```
1. Init.init()
   └─ Apache Santuario must be initialized once before any canonicalization

2. ReadConfig → config.properties → mtsa-pilot.properties / mtsa.properties
   └─ Loads: kspath, kspass, DB_URL/USER/PASS, jsonAuth, service, debug

3. Credential.AuthenticateHdr(request)
   └─ Basic Auth → decode → validate against credential.json array

4. Parse JSON body: { "base64xml": "..." }
   └─ XmlFileReader.decodeBase64Xml(base64xml) → xmlString

5. SignUsingP12(kspath, kspass)
   └─ Load PKCS12 → extract PrivateKey + X509Certificate + chain
   └─ Auto-detect: RSA → SHA256withRSA | EC → SHA256withECDSA

6. XmlHandler(xmlString, canonAlgoUri)
   └─ Parse XML DOM, extract <Object Id="ObjectContent"> element
   └─ objectElement.setIdAttribute("Id", true)  ← important for URI reference

7. XmlHandler.updateXMLElement(null)
   └─ Set algorithm URIs on CanonicalizationMethod, SignatureMethod, DigestMethod
   └─ Clear <SignatureValue>
   └─ removeUnusedXsiNamespace() — strips xmlns:xsi if not used

8. XmlDigest.getDocDigest(objectElement)
   └─ Canonicalize <Object> with EXC-C14N (Apache Santuario Canonicalizer)
   └─ SHA-256 → Base64 → DigestValue string

9. XmlHandler.updateXMLElement(digestValue)
   └─ Set DigestValue text content in DOM
   └─ Returns updated signedInfoElement

10. sig.signSignedInfo(signedInfoElement, canonAlgoUri)
    └─ If namespace null → set xmlns="http://www.w3.org/2000/09/xmldsig#"
    └─ Canonicalize <SignedInfo> with EXC-C14N
    └─ Signature.initSign(privateKey).update(canonBytes).sign()
    └─ Base64 → SignatureValue string

11. XmlHandler.updateXMLValue(latestXml, signatureValue, certChain, signedInfo)
    └─ Re-parse XML, replace <SignedInfo> node (importNode), set <SignatureValue>
    └─ Serialize DOM → compact string (no indent, UTF-8, standalone)

12. ConvertUtils.convertFinalXmlStrToBase64(finalXml)
    └─ Save file + db.saveTxDb(txid, digestValue, signatureValue, status)

13. Return: { statusCode:"000", txID, signedData (base64), certX509, subjectDN }
```

---

## Verification Flow (verify.java)

```
Input: { "signedXmlBase64": "..." }

1. Decode base64 → signed XML string
2. XmlHandler.XmlElementExtractor(xmlStr)
   └─ XPath extract: DigestValue, SignatureValue, algorithms, Reference URI
   └─ Returns Signature DTO

3. verifyHash:
   p12.verifyHash(objectElement, digestValue, canonAlgoUri)
   └─ Canonicalize <Object> → SHA-256 → Base64
   └─ Compare with extracted DigestValue

4. verifySignature:
   p12.verifySignedInfoSignature(signedInfoEl, signatureValue, canonAlgoUri)
   └─ Canonicalize <SignedInfo> → verify RSA-SHA256 against P12 public key

Status codes:
  "000" → both hash and signature valid
  "121" → digest mismatch (content tampered)
  "122" → signature invalid (wrong key or tampered SignedInfo)
```

---

## Key Classes

| Class | Package | Purpose |
|-------|---------|---------|
| `sign.java` | `mytrustsigner` | POST /sign servlet |
| `verify.java` | `mytrustsigner` | POST /verify servlet |
| `getcertinfo.java` | `mytrustsigner` | GET /getcertinfo servlet |
| `XmlHandler.java` | `xmlhandler` | Core DOM manipulation + C14N |
| `SignUsingP12.java` | `xmlhandler` | PKCS12 load, sign, verify |
| `ReadConfig.java` | `xmlhandler` | 2-tier config loader |
| `XmlDigest.java` | `xmlhandler` | Object element digest |
| `Signature.java` | `xmlhandler` | XMLDSig elements DTO |
| `XmlFileReader.java` | `xmlhandler` | Base64 XML decode + file hash |
| `Credential.java` | `authentication` | Basic Auth + JSON credential file |
| `DBUtil.java` | `util` | MySQL TXID + transaction save |
| `DatabaseConfig.java` | `util` | JDBC connection factory |
| `CanonicalizationAlgo.java` | `util` | C14N algo enum (default: EXC-C14N) |
| `ConvertUtils.java` | `util` | Base64 utils, file save |
| `Debugging.java` | `util` | Config-controlled debug logger |

---

## Canonicalization Algorithms

```java
// Enum in CanonicalizationAlgo.java
EXC_C14N = "http://www.w3.org/2001/10/xml-exc-c14n#"            // DEFAULT — exclusive C14N
C14N_OMIT_COMMENTS = "http://www.w3.org/TR/2001/REC-xml-c14n-20010315"
C14N_WITH_COMMENTS = "http://www.w3.org/TR/2001/REC-xml-c14n-20010315#WithComments"

// Usage with Apache Santuario
Init.init();  // MUST call before any canonicalization
Canonicalizer canon = Canonicalizer.getInstance(canonAlgoUri);
ByteArrayOutputStream bos = new ByteArrayOutputStream();
canon.canonicalizeSubtree(element, bos);
byte[] canonBytes = bos.toByteArray();
```

**Why EXC-C14N?** Exclusive C14N only includes namespace declarations that are
visibly utilized — avoids namespace "pollution" from ancestor elements, making
signatures stable across XML transformations.

---

## Config System

### Tier 1 — WEB-INF/classes/config.properties
```properties
app.config=/opt/miti/          # base path (trailing slash required)
app.stage=pilot                 # pilot | production
```

### Tier 2 — External properties file
```
Pilot:      {app.config}properties/mtsa-pilot.properties
Production: {app.config}properties/mtsa.properties
```

```properties
# mtsa.properties (production)
kspath=/opt/miti/certs/miti-signing.p12    # PKCS12 keystore path
kspass=yourP12Password                      # PKCS12 password
DB_URL=jdbc:mysql://host:3306/dbname
DB_USER=dbuser
DB_PASSWORD=dbpass
jsonAuth=credential.json                    # filename only (resolved relative to properties/)
service=200                                 # 200=OK, 503=maintenance mode
debug=false
pkiws_account=miti_prod
stage=production
```

### Auth Credential File
Path: `{app.config}properties/{jsonAuth}`
```json
[
  { "username": "miti_client", "password": "secretpass" },
  { "username": "another_client", "password": "anotherpass" }
]
```

---

## Database Schema

```sql
CREATE TABLE transactions (
  id              BIGINT AUTO_INCREMENT PRIMARY KEY,
  tx_id           VARCHAR(30),     -- epoch millis + microseconds (unique)
  digest_value    TEXT,            -- base64 SHA-256 of canonicalized Object
  signature_value TEXT,            -- base64 RSA-SHA256 of canonicalized SignedInfo
  status          VARCHAR(50),     -- "Success" | "Failed" | error message
  tx_timestamp    DATETIME         -- NOW()
);
```

TXID generation:
```java
long currentTimeMillis = System.currentTimeMillis();
long nanoTime = System.nanoTime();
long microTime = (nanoTime / 1_000L) % 1_000_000L;
String txid = currentTimeMillis + String.format("%06d", microTime);
// Example: "1716534012345123456"
```

---

## API Contracts

### POST /sign
```json
Request headers:
  Authorization: Basic base64(username:password)
  Content-Type: application/json

Request body:
  { "base64xml": "PD94bWwgdmVyc2..." }

Response (success):
{
  "statusCode": "000",
  "statusMsg": "Signing data success",
  "txID": "1716534012345123456",
  "signedData": "PD94bWwgdmVyc2...",   // base64 signed XML
  "signedDataHash": "abc123...",         // SHA-256 hex of saved signed file
  "certX509": "MIIFxDCC...",            // base64 DER cert
  "subjectDN": "CN=MITI Signing Cert, O=MITI, C=MY"
}

Response (error):
  { "statusCode": "101", "statusMsg": "Error parsing JSON request string" }
  { "statusCode": "102", "statusMsg": "Signing failed" }
```

### POST /verify
```json
Request body:
  { "signedXmlBase64": "PD94bWwgdmVyc2..." }

Response:
{
  "statusCode": "000",       // 000=valid, 121=digest fail, 122=signature fail
  "statusMsg": "XML document is valid",
  "verifyHash": true,
  "verifySignature": true
}
```

### GET /getcertinfo
```json
Response (success):
{
  "statusCode": "000",
  "statusMsg": "Success",
  "certStatus": "valid",          // valid | expired | not yet valid
  "certValidFrom": "...",
  "certValidTo": "...",
  "certSerialNo": "123456...",
  "certSubjectDN": "CN=...",
  "certIssuer": "CN=...",
  "certX509": "MIIFxDCC...",
  "certpubKey": "MIIBIjAN..."
}
```

---

## Production Deployment Checklist

```
[ ] 1. Create /opt/miti/properties/ directory on prod server
[ ] 2. Create mtsa.properties with correct prod values (kspath, kspass, DB creds)
[ ] 3. Place prod .p12 signing cert at kspath
[ ] 4. Create credential.json with authorized client credentials
[ ] 5. Create MySQL transactions table (DDL above)
[ ] 6. Update WEB-INF/classes/config.properties → app.stage=production
        OR: use correct config.properties that already points to prod
[ ] 7. Build WAR: mvn clean package
        → target/MyTrustSignerXML-1.0-SNAPSHOT.war
[ ] 8. Deploy WAR to Tomcat webapps/ (rename if needed for context path)
[ ] 9. Test sequence:
        a. GET /getcertinfo → verify cert loaded + valid
        b. POST /sign with SANDBOX-API-ORINGINAL.xml base64 → get signedData
        c. POST /verify with signedData → statusCode 000
```

---

## Common Issues and Root Causes

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| `NullPointerException` in `ReadConfig` | `config.properties` not found in classpath | Ensure file is at `WEB-INF/classes/config.properties` in WAR |
| `FileNotFoundException` loading properties | `app.config` path wrong or trailing slash missing | Check path format: `/opt/miti/` not `/opt/miti` |
| `KeyStoreException` loading P12 | Wrong password, wrong P12 type, or corrupted file | Test with `keytool -list -keystore file.p12 -storetype PKCS12` |
| `401 Unauthorized` | Username not in credential.json, or wrong password | Check JSON format: `[{"username":"...","password":"..."}]` |
| `503 Service Unavailable` | `service=503` in properties | Change `service=200` in mtsa.properties |
| `DigestValue mismatch` in verify | XML was modified after signing | Content integrity violated — re-sign |
| `NullPointerException` on `db.saveTxDb` | DB connection failed + saveTxDb called in catch block | Fix: null-check `db` before calling saveTxDb in error handlers |
| Namespace mismatch after parsing | DOM serializer adds unwanted namespace declarations | Check `removeUnusedXsiNamespace()` runs before signing |
| Signing works but verify fails | EXC-C14N not applied consistently in verify | Ensure same canonAlgoUri used in both sign and verify |

---

## Dev vs Prod Path Layout

```
Dev (source):
  C:\PROJECTS\MITI\Development\MyTrustSignerXML\
  ├── src/main/java/com/msctg/miti/mytrustsigner/
  ├── webapps/SandboxAPI2#MyTrustSignerXMLPilot/  ← exploded sandbox WAR
  ├── pom.xml
  └── (no config/ — config lives in external dir on server)

Prod server expected layout:
  /opt/miti/
  ├── properties/
  │   ├── mtsa.properties           ← prod config
  │   ├── mtsa-pilot.properties     ← pilot config
  │   └── credential.json           ← auth users
  ├── certs/
  │   └── miti-signing.p12          ← prod signing cert
  └── files/                        ← signed XML files saved here
      └── {txid}                    ← one file per transaction

Tomcat webapps:
  └── {ContextPath}/                ← deployed WAR
      └── WEB-INF/classes/config.properties → points to /opt/miti/
```

---

## Sample Files

### Original (unsigned) XML
Located: `C:\PROJECTS\MITI\TestSignXML\SANDBOX-API-ORINGINAL.xml`
- XMLDSig envelope with empty DigestValue, SignatureValue, no Algorithm attributes
- `<Object Id="ObjectContent">` contains `<Package>` MITI payload
- Reference URI: `#ObjectContent`

### Signed XML
Located: `C:\PROJECTS\MITI\TestSignXML\SANDBOX-API-SIGNED.xml`
- Same structure with DigestValue, SignatureValue, and Algorithm URIs populated
- Use this as reference for expected output format

---

## Level History
- **Lv.1** — Base: Full XMLDSig signing + verify flow, Apache Santuario usage, config system,
  auth system, DB schema, API contracts, production deployment checklist, common issues.
  (Origin: 2026-05-24 — studied from MyTrustSignerXML-MITI project source code)
