---
name: signing-labs
description: >
  Use this skill for ALL tasks in the MyTrustPDFSigner_IT5 signing lab project —
  creating new hash preparation classes, embed signature classes, writing test runners,
  debugging iText5 signing, or understanding the deferred signing flow.
  Trigger when user says "signing lab", "signing exercise", "signing test",
  "new hash class", "new signing scenario", "PDF_prepareHash", "PDF_embedSignature",
  or mentions path "MyTrustPDFSigner_IT5".
  Also triggers on: "sign with image", "sign with form field", "multi-party signing",
  "MPS", "sign level", "sigImage", or any iText5 deferred signing topic.
---

# Signing Labs — MyTrustPDFSigner_IT5
*Hands-on iText5 PDF deferred signing lab with MSC Trustgate PKIWS integration*

## Activation

When this skill activates:
`🧪 Signing Labs loaded — MyTrustPDFSigner_IT5 ready.`

Then apply the knowledge below to assist with the task.

---

## Project Location

```
C:\PROJECTS\SIGNING LABS\MyTrustPDFSigner_IT5\
```

---

## Package Structure

```
src/main/java/
├── com.msctg.mytrustsigner.hash          ← Hash preparation (Phase 1) — ADD NEW CLASSES HERE
│   ├── PDF_prepareHash.java              — Basic: coordinates, DESCRIPTION mode
│   ├── PDF_prepareHash_Image.java        — With sig image, GRAPHIC mode
│   ├── PDF_prepareHash_Image_MPS.java    — Multi-party: image + text form field, sign_level
│   └── PDF_prepareHash_signField.java    — Existing sig field by keyword/field name
│
├── com.msctg.mytrustpdfsigner           ← Embed signature (Phase 3)
│   ├── PDF_embedSignature.java           — Basic embed with TSA
│   ├── PDF_embedSignature_MPS.java       — Multi-party embed
│   └── PDF_embedSignature_signField.java — Embed for sign field variant
│
├── com.msctg.mytrustpdfsigner.formfield ← PDF form field helpers
│   ├── PdfFormFieldHelper.java
│   ├── PdfFieldNameValue.java
│   ├── PDF_convertField.java
│   └── KeyWordHelper.java
│
└── com.msctg.test                       ← End-to-end test runners
    ├── signWithRoaming_image_only.java
    ├── signWithRoaming_sigImage_and_fftext.java
    ├── signWithRoaming_sigImage_KeywordWithQR.java
    ├── signWithKey.java
    ├── signWithMobile.java
    └── PdfFormFieldChecker.java

src/test/java/
    ├── FileDigest.java
    ├── FileToBase64Converter.java
    └── GetDocumentHash.java

src/wsdl/                                ← JAX-WS stubs for PKIWS + RoamingWS
    digitalid.msctrustgate.com/pkiws_pilot/
    digitalid.msctrustgate.com/pkiws_prod/
```

---

## Signing Flow (Two Phases)

```
Phase 1 — prepareHash (in com.msctg.mytrustsigner.hash)
  Input : src PDF, signer X.509 cert, visibility params
  Action: open PDF → place blank signature container (8192 bytes reserved)
          compute SHA256(ByteRange)  → digest
          build PdfPKCS7 → getAuthenticatedAttributeBytes() → sh (signedAttrs)
          SHA256(sh)                 → hash (for NONEwithECDSA/NONEwithRSA)
          capture Calendar.getTimeInMillis() → timestamp
  Output: JSON { "digest": "...", "hash": "...", "timestamp": "..." }
          → saves temp PDF to disk

Phase 2 — signPKIWS (SOAP call to PKIWS RoamingWS)
  Input : hash (base64), certSerial, pin, account, stage
  Action: POST to roamingws → MSC Trustgate signs hash with signer's private key
  Output: extSignature_b64 (base64 raw ECDSA or RSA signature)

Phase 3 — embedSignature (in com.msctg.mytrustpdfsigner)
  Input : temp PDF, extSignature_b64, digest, chain, timestamp
  Action: rebuild PdfPKCS7 with same timestamp
          setExternalDigest(extSignature, null, "ECDSA" or "RSA")
          optionally add TSA via TSAClientBouncyCastle
          getEncodedPKCS7(digest_bytes, cal, tsc) → CMS blob
          MakeSignature.signDeferred(reader, sign_name, os, external)
  Output: final signed PDF saved to dest
```

---

## JSON Contract (prepareHash output)

```json
{
  "digest": "<base64 of SHA256(ByteRange)>",
  "hash":   "<base64 of SHA256(signedAttributes)>",
  "timestamp": "<Calendar.getTimeInMillis() as long>"
}
```

- `digest` → passed to `embedSignature` as-is
- `hash` → sent to PKIWS for signing (use `NONEwithECDSA` / `NONEwithRSA`)
- `timestamp` → MUST pass same value back to `embedSignature` to rebuild identical signedAttributes

---

## Existing Hash Class Comparison

| Class | Sig Placement | Rendering Mode | Extra Params | Use Case |
|-------|--------------|----------------|--------------|----------|
| `PDF_prepareHash` | Coordinates (x1,y1,x2,y2) | DESCRIPTION | — | Basic text signature |
| `PDF_prepareHash_Image` | Coordinates (x1,y1,x2,y2) | GRAPHIC | `sigImage` (path to image) | Signature image only |
| `PDF_prepareHash_Image_MPS` | Existing field by name `signature_{sign_level}` | GRAPHIC | `sigImage`, `sign_level` | Multi-party + form field text update |
| `PDF_prepareHash_signField` | Existing field by name `signature_{keyword}` | GRAPHIC | `keyword`, `sigImage` | Pre-existing sig field in PDF |

---

## How to Add a New Hash Class

When a new signing scenario requires different appearance or placement, create a new class in:
```
com.msctg.mytrustsigner.hash
```

### Template pattern

```java
package com.msctg.mytrustsigner.hash;

// ... imports (same as PDF_prepareHash_Image) ...

public class PDF_prepareHash_YourVariant {

    public static String emptySignature_hash(String src, String dest,
            Certificate[] chain, /* your extra params */ ) throws ... {
        JSONObject obj = new JSONObject();

        PdfReader reader = new PdfReader(src);
        FileOutputStream os = new FileOutputStream(dest);
        PdfStamper stamper = PdfStamper.createSignature(reader, os, '\000', null, true);
        PdfSignatureAppearance appearance = stamper.getSignatureAppearance();

        // 1. Determine sign_name
        AcroFields acroFields = reader.getAcroFields();
        List<String> signatureNames = acroFields.getSignatureNames();
        String sign_name = "signature_" + (signatureNames.size() + 1); // or by field/keyword

        // 2. Set appearance (customize here per scenario)
        if (visibility) {
            appearance.setVisibleSignature(new Rectangle(x1, y1, x2, y2), page, sign_name);
            // add image, text, rendering mode as needed
            appearance.setRenderingMode(PdfSignatureAppearance.RenderingMode.GRAPHIC);
        }
        appearance.setCertificate(chain[0]);

        // 3. Reserve blank signature space
        ExternalSignatureContainer external = new ExternalBlankSignatureContainer(
                PdfName.ADOBE_PPKLITE, PdfName.ADBE_PKCS7_DETACHED);
        MakeSignature.signExternalContainer(appearance, external, 8192);

        // 4. Compute hash
        InputStream inp = appearance.getRangeStream();
        BouncyCastleDigest digest = new BouncyCastleDigest();
        byte[] hash = DigestAlgorithms.digest(inp, digest.getMessageDigest("SHA256"));

        Calendar cal = Calendar.getInstance();
        PdfPKCS7 sgn = new PdfPKCS7(null, chain, "SHA256", null, digest, false);
        byte[] sh = sgn.getAuthenticatedAttributeBytes(hash, cal, null, null,
                MakeSignature.CryptoStandard.CMS);

        MessageDigest messageDigest = MessageDigest.getInstance("SHA256");
        messageDigest.update(sh);
        byte[] hash_byte = messageDigest.digest();

        // 5. Return JSON
        obj.put("digest", Base64.getEncoder().encodeToString(hash));
        obj.put("hash", Base64.getEncoder().encodeToString(hash_byte));
        obj.put("timestamp", cal.getTimeInMillis());
        return obj.toString();
    }
}
```

### Key differences per scenario

| Scenario | What changes in prepareHash |
|----------|-----------------------------|
| Text description only | `RenderingMode.DESCRIPTION`, `setLayer2Text(...)` |
| Image only | `RenderingMode.GRAPHIC`, `setSignatureGraphic(image)` |
| Image + text overlay | `RenderingMode.GRAPHIC_AND_DESCRIPTION` |
| Existing sig field | `appearance.setVisibleSignature(sign_name)` — no Rectangle |
| Multi-party | `sign_name = "signature_" + sign_level`, update form fields before signing |
| QR code | Add QR image to `PdfContentByte` over/under layer before hash computation |

---

## Test Runner Pattern

Test classes live in `com.msctg.test`. Each runner does:

```java
// 1. Set file paths
String SRC  = "C:\\PROJECTS\\SIGNING LABS\\file\\src\\testPDF.pdf";
String TEMP = "C:\\PROJECTS\\SIGNING LABS\\file\\temp\\testPDF_temp.pdf";
String DEST = "C:\\PROJECTS\\SIGNING LABS\\file\\final\\testPDF_DS.pdf";

// 2. Get certificate from PKIWS
String x509b64 = getX509Cert(stage, account, certType, certSerial, idNo);
// → builds chain[]

// 3. Prepare hash
String prephash = PDF_prepareHash_YourVariant.emptySignature_hash(SRC, TEMP, chain, ...);
JSONObject obj_prephash = new JSONObject(prephash);
String digest    = obj_prephash.getString("digest");
String hash      = obj_prephash.getString("hash");
String timestamp = obj_prephash.getString("timestamp");

// 4. Sign via PKIWS
String extSignature_b64 = signPKIWS(stage, account, idNo, certSerial, useHsm, pin, hash);

// 5. (Optional) Verify locally
Signature dsa = Signature.getInstance("NONEwithECDSA");
dsa.initVerify(chain[0]);
dsa.update(Base64.getDecoder().decode(hash));
boolean isValid = dsa.verify(Base64.getDecoder().decode(extSignature_b64));

// 6. Embed signature
PDF_embedSignature.createSignature(TEMP, DEST, digest, extSignature_b64, chain, timestamp);
```

---

## PKIWS Connection (Pilot vs Production)

| Stage | WS URL | Account | Notes |
|-------|--------|---------|-------|
| pilot | `https://digitalid.msctrustgate.com/pkiws_pilot/v3/pkiws?wsdl` | `mobilepkiECC_pilot` | Headers: root/12345678 |
| production | `https://digitalid.msctrustgate.com/pkiws/v3/pkiws?wsdl` | `aatl_prod` | Headers: msctgra/*** |
| TSA | `https://digitalid.msctrustgate.com/adss/tsa` | null | Used in embedSignature |

JAX-WS stubs generated from `src/wsdl/` → `target/generated-sources/jaxws-wsimport/digitalid/`
- Pilot: `digitalid.pkiws_pilot.pkiws.*`, `digitalid.pkiws_pilot.roaming.*`
- Prod: `digitalid.pkiws_prod.pkiws.*`, `digitalid.pkiws_prod.roaming.*`

---

## File Paths (Lab Files)

```
C:\PROJECTS\SIGNING LABS\file\src\     ← source PDFs + signature images
C:\PROJECTS\SIGNING LABS\file\temp\    ← temp PDFs (after prepareHash)
C:\PROJECTS\SIGNING LABS\file\final\   ← final signed PDFs
C:\PROJECTS\SIGNING LABS\file\src\MPS\ ← MPS-specific fonts/assets (Arial.ttf)
```

---

## Common Issues

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| Adobe "Document altered" | `timestamp` not passed correctly to embedSignature — signedAttrs rebuilt with wrong time | Ensure same `cal.getTimeInMillis()` used in both phases |
| Signature invalid (local verify) | Wrong algo — used SHA256withECDSA but hash is already SHA256'd | Use `NONEwithECDSA` since `hash` is already `SHA256(signedAttrs)` |
| `ExceptionConverter: Not enough space` | `8192` bytes reservation too small for large cert chains | Increase to `16384` or `32768` in `signExternalContainer(appearance, external, 8192)` |
| `PdfException: Field 'signature_X' not found` | sign_name doesn't match existing field in PDF | Check field names with `PdfFormFieldChecker` or `AcroFields.getFields().keySet()` |
| Form field text not flattened | Forgot `stamper.setFormFlattening(true)` in MPS variant | Add before `signExternalContainer` call |
| Image distorted in sig field | Not using `setImageScale(-3)` or image dimensions mismatch | Use `appearance.setImageScale(-3)` for proportional scaling |

---

## Level History

- **Lv.1** — Base: Full project structure, two-phase flow, hash class patterns, test runner pattern, new class template, common issues.
  (Origin: 2026-05-19, built from reading real project source at C:\PROJECTS\SIGNING LABS\MyTrustPDFSigner_IT5)
