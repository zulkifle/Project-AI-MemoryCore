---
name: mygpki-signing
description: >
  MUST use when working on MyGPKI INTERNAL signing flow, MTSA signing platform,
  PDF digital signing, iText5 deferred signing, CMS SignedData, PdfSigningService,
  sign_roaming, sign_roaming_next, SignCert, SignComplete, or any task involving
  MyGPKI Roaming (OTP, cert-first flow). Also triggers on: "signing issue",
  "Adobe invalid", "document altered", "hashBase64", "signedAttributes",
  "ECDSA signing", "CMS embedding", or "MyGPKI JS SDK" in context of MTSA/JKR project.
---

# MyGPKI INTERNAL Signing — Expert Skill
*Deep knowledge of the MTSA cert-first deferred PDF signing flow using MyGPKI Roaming*

## Activation

When this skill activates:
`🔐 MyGPKI Signing Skill loaded — cert-first deferred signing flow ready.`

Then apply the knowledge below to assist with the task.

---

## The Three-Phase Signing Flow

INTERNAL signing uses a **cert-first deferred signing** architecture — the signer's
X.509 certificate must be known BEFORE the signing hash is computed, because the
certificate is included in the CMS signedAttributes structure.

```
Phase 1 — preparePdfForSigning()
  Input : raw PDF bytes, fieldName, signKeyword, signerInfo
  Action: locate keyword → place sig field → reserve /Contents space (32768 bytes)
          compute SHA256(ByteRange) → messageDigest
          store: preparedPdf, messageDigest, signingTimeMillis in session
  Output: PrepareResult (preparedPdf, messageDigest, signingTimeMillis)

Phase 2 — computeHashWithCert()   ← triggered by POST /api/sign/cert
  Input : preparedPdf, fieldName, signingTimeMillis, signerCertBase64 (from MyGPKI)
  Action: parse X.509 cert
          build signedAttributes = DER( content-type | messageDigest | signing-time )
          return Base64(signedAttrs bytes) as hashBase64
  Output: hashBase64 → browser → sign_roaming_next(gpkiSessionId, seq, hashBase64)

Phase 3 — embedInternalCmsSignature()   ← triggered by POST /api/sign/complete
  Input : preparedPdf, fieldName, rawEcdsaSig (from MyGPKI), signerCertBase64,
          messageDigestBytes, signingTimeMillis
  Action: rebuild identical signedAttributes using stored messageDigest + signingTime
          call PdfPKCS7.setExternalDigest(rawEcdsaSig, null, "ECDSA")
          call PdfPKCS7.getEncodedPKCS7() → CMS SignedData blob
          embed CMS into prepared PDF /Contents via MakeSignature.signDeferred()
  Output: fully signed PDF bytes
```

---

## CRITICAL: MyGPKI SHA256withECDSA Behavior

**MyGPKI `sign_roaming_next()` uses SHA256withECDSA internally.**
It hashes the `data` parameter with SHA-256 before signing with ECDSA.

```
CORRECT (v1.4.1+):
  We send:    signedAttributes bytes (raw DER, not pre-hashed)
  MyGPKI:     SHA256(signedAttrs) + ECDSA  →  ECDSA( SHA256(signedAttrs) )
  Adobe:      SHA256(signedAttrs) + verify ECDSA  →  ✓ VALID

WRONG (v1.4.0 and earlier — DO NOT DO THIS):
  We send:    SHA256(signedAttrs)  ← pre-hashed
  MyGPKI:     SHA256(SHA256(signedAttrs)) + ECDSA  →  double-hash
  Adobe:      SHA256(signedAttrs) + verify ECDSA  →  ✗ "Document altered"
```

**Rule:** Never pre-hash the signedAttributes before passing to sign_roaming_next.
The hash step is MyGPKI's responsibility, not ours.

---

## Key Classes and Files

| Class / File | Location | Purpose |
|---|---|---|
| `PdfSigningService.java` | `src/java/my/gov/gpki/service/` | Core signing logic — 3 phases |
| `SignInit.java` | `src/java/my/gov/gpki/servlet/` | POST /api/sign/init — Phase 1 |
| `SignCert.java` | `src/java/my/gov/gpki/servlet/` | POST /api/sign/cert — Phase 2 |
| `SignComplete.java` | `src/java/my/gov/gpki/servlet/` | POST /api/sign/complete — Phase 3 |
| `signing.jsp` | `web/jsp/` (dev) / `SandboxAPI#MTSAPilotJKR/jsp/` (deploy) | Browser-side MyGPKI JS interaction |
| `SigningSession.java` | `src/java/my/gov/gpki/model/` | Session state — stores preparedPdf, messageDigest, signingTimeMillis, signerCert |
| `AppConfig.java` | `src/java/my/gov/gpki/config/` | `cmsReservedSize`, `environmentMode`, `signingDebugLog` |

---

## Key Methods in PdfSigningService

```java
// Phase 1
PrepareResult preparePdfForSigning(byte[] pdfBytes, String fieldName,
    String signKeyword, SignerInfo signerInfo)

// Phase 2 — returns Base64(signedAttrs bytes) — sent to sign_roaming_next
String computeHashWithCert(byte[] preparedPdf, String fieldName,
    long signingTimeMillis, String signerCertBase64)

// Phase 3 — embeds ECDSA sig + cert into CMS, embeds into prepared PDF
byte[] embedInternalCmsSignature(byte[] preparedPdf, String fieldName,
    String rawSignatureBase64, String signerCertBase64,
    byte[] messageDigestBytes, long signingTimeMillis)

// Diagnostic — BouncyCastle local verification
List<VerifyResult> verifySignature(byte[] pdfBytes)
```

---

## Adobe Signature Validation Chain

For Adobe to show a signature as **valid**, ALL three must pass:

```
1. Content integrity:
   SHA256( PDF bytes at /ByteRange offsets ) == messageDigest in CMS signedAttrs

2. Signature integrity:
   ECDSA.verify( SHA256(signedAttrs), ecdsaSignature, signerPublicKey ) == true

3. Certificate trust:
   signerCert → intermediate CA → root CA  (must be in Adobe trusted store)
```

Failure symptoms:
- "Document has been altered or corrupted" = Step 1 or 2 failed
- "Signer's identity is unknown" = Step 3 failed (cert not trusted)
- "Signer's identity is valid" + "Document altered" = Step 1 or 2 failed, NOT cert trust

---

## Diagnostic Logging

Enable verbose logging by setting in `gpki.properties`:
```
gpki.signing.debugLog=true
```

Log prefixes to look for:
- `[DIAG-P1]` — Phase 1: ByteRange, messageDigest, signingTime
- `[CERT-HASH]` — Phase 2: cert details, signedAttrs, hashToSign
- `[EMBED-BC]` — Phase 3: CMS build, signedAttrs match check, cmsSize
- `[VERIFY-LOCAL]` — Post-sign BouncyCastle verification result in SignComplete

Key diagnostic check in `[EMBED-BC]`:
```
[EMBED-BC] SHA256(attrs) : <value>  ← MUST match [CERT-HASH] hashToSign
```
If these don't match, the signing time or messageDigest is inconsistent between Phase 2 and Phase 3.

---

## Browser Flow (signing.jsp)

```
1. Page loads → reads SESSION_ID from URL param
2. Calls POST /api/sign/init (already done by caller) → session exists
3. User clicks "Tandatangan" → calls gpki_requestOTP()
4. MyGPKI JS: request_otp_roaming(userId, gpkiCallback)
5. User enters PIN + OTP → gpki_submitOTP()
6. MyGPKI JS: sign_roaming(gpkiSessionId, seq, hash, cert, gpkiCallback)
   → dummy call to get signer cert
7. Callback: calls POST /api/sign/cert { sessionId, signerCertBase64 }
   → receives hashBase64 (signedAttrs bytes)
8. Calls sign_roaming_next(gpkiSessionId, seq, hashBase64)
   → MyGPKI signs hashBase64 with SHA256withECDSA
   → callback receives cmsSignatureBase64 (raw ECDSA DER sig)
9. Calls POST /api/sign/complete { sessionId, cmsSignatureBase64 }
   → Phase 3 embeds CMS → PDF saved
10. Shows success page
```

---

## MyGPKI Code 25 — OTP Still Valid

When MyGPKI returns `status: false` during OTP request and the message contains
"masih sah" — this means the OTP is still valid from a previous request.
The response has `code: null` (NOT `code: '25'`).

Fix in `signing.jsp` — detect by message text, not code value:
```javascript
if (code === '25' || (result.message && result.message.indexOf('masih sah') !== -1)) {
    // OTP still valid — restore nonce from sessionStorage and resume OTP section
    showSection('sectionOtp');
    goToStep(2);
    startCountdown(parseInt(OTP_EXPIRY) || 300);
    showOtpBanner('OTP anda masih sah. Sila masukkan PIN dan OTP...', 'info');
}
```

---

## Common Issues and Root Causes

| Symptom | Root Cause | Fix |
|---|---|---|
| Adobe "Document altered" + "Identity valid" | Double SHA256 — pre-hashing before sign_roaming_next | Send raw signedAttrs bytes, not SHA256(signedAttrs) |
| Adobe "Document altered" + "Identity unknown" | Cert not in Adobe trust store | Install MyGPKI root CA in Adobe trusted certs |
| `[EMBED-BC] SHA256(attrs)` ≠ `[CERT-HASH] hashToSign` | signingTime or messageDigest inconsistency between Phase 2 and Phase 3 | Ensure session stores exact same values used in both phases |
| OTP page shows error instead of OTP section | Code 25 not detected (result.code is null, not '25') | Detect by message.indexOf('masih sah') |
| CMS exceeds reserved space | cmsReservedSize too small | Increase `gpki.signing.cmsReservedSize` (default 32768) |
| "Keyword not found in PDF" | signKeyword text not present in PDF | Check PDF template has correct keyword text |

---

## Deployment Sync Rule

When changing signing code:
1. Edit **development source first**: `C:\PROJECTS\MyGPKI-SKALA\Development\mtsa\`
2. Then sync to **deployment copy**: `C:\PROJECTS\MyGPKI-SKALA\Deployment\PILOT\SandboxAPI#MTSAPilotJKR\`
3. For config files (gpki.properties, etc.): update deployment copy at
   `C:\PROJECTS\MyGPKI-SKALA\Deployment\PILOT\mtsa_jkr\config\`

---

## Level History

- **Lv.1** — Base: Core 3-phase flow, SHA256withECDSA fix, diagnostic logging, OTP code 25 handling.
  (Origin: 2026-03-27, from live debugging session — "Document altered" root cause investigation)
