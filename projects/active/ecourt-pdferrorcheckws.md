# ECOURT_PdfErrorCheckWS - PDF Error Check SOAP Web Service
*JAX-WS SOAP service on WildFly to check and auto-repair PDFs before digital signing*

## Project Overview
- **Type**: API (SOAP Web Service)
- **Client**: Internal (POJ / e-Court)
- **Period**: 2026-07-02 - Active
- **Tech Stack**: Java 8 + JAX-WS + WildFly 18 + iText5 5.4.1 + log4j
- **Completion**: 98%
- **Duration**: ~4 hours
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-07-28 — Fixed false-negative: checker wasn't catching a ClassCastException that only surfaces during real signing
- **Next Steps**:
  1. Run `testCheck.java` against the actual offending PDF to confirm it now returns 100/failed
  2. Clean and Build in NetBeans, redeploy WAR to WildFly
- **Known Issues**:
  - `ClassCastException: PdfArray cannot be cast to PdfDictionary` → 100/failed, no auto-fix (AcroForm/Annots structure corruption — not fixable without external tools). Previously this could slip through as 000/success on some PDFs because the check didn't reach the code path where it occurs — see 2026-07-28 session.
  - `InvalidPdfException: Rebuild failed: trailer not found` → 100/failed, too corrupt for iText

## SOAP Methods

| Method | Params | Purpose |
|--------|--------|---------|
| `CheckPdfError` | `PdfBase64`, `FileName` | Check PDF from Base64; auto-rp xref damage in memory |
| `CheckPdfErrorByPath` | `PDFtoSign`, `gUid` | Check PDF from file path; auto-rp xref damage, overwrite file |

## Response Format
```xml
<errCode>000</errCode>   <!-- 000=success, 100=failed -->
<status>success</status> <!-- success / failed -->
<errMsg>pdf verification pass</errMsg>
```

## Error Scenarios & Handling

| Error | Auto-fix? | Client Response |
|-------|-----------|-----------------|
| Clean PDF | — | 000 / success / pdf verification pass |
| `isRebuilt() = true` (xref damage) | Yes — PdfStamper | 000 / success / pdf verification pass |
| `ClassCastException` (AcroForm corrupt) | No | 100 / failed / full exception message |
| `InvalidPdfException: Rebuild failed` | No — too corrupt | 100 / failed / full exception message |

## Project Structure
```
ECOURT_PdfErrorCheckWS/
├── src/java/com/poj/pdferrorcheck/
│   ├── PdfErrorCheckService.java   ← @WebService, 2 methods
│   ├── PdfErrorCheckHelper.java    ← logic: isRebuilt auto-rp + deep signature-placeholder check
│   └── PdfCheckResult.java         ← errCode, status, errMsg
├── web/WEB-INF/
│   ├── jboss-web.xml               ← context-root: /PdfErrorCheckWS
│   └── web.xml                     ← minimal (WildFly handles @WebService)
├── nbproject/
│   ├── project.xml                 ← libs: log4j + itextpdf-5.4.1
│   ├── project.properties          ← JDK8, WildFly, war.name=PdfErrorCheckWS.war
│   └── jax-ws.xml                  ← registers PdfErrorCheckService
└── dist/PdfErrorCheckWS.war
```

## Technical Notes
- **Repository**: `C:\PROJECTS\ECOURT\WebServiceProject\POJ\ECOURT_PdfErrorCheckWS`
- **WAR name**: `PdfErrorCheckWS.war`
- **Context root**: `/PdfErrorCheckWS`
- **WSDL**: `http://<server>:<port>/PdfErrorCheckWS/PdfErrorCheckService?wsdl`
- **targetNamespace**: `http://signing/`
- **Key Dependencies**: `itextpdf-5.4.1.jar`, `log4j.jar` — all in `C:\PROJECTS\java Library\`
- **iText5 ExceptionConverter**: iText5 wraps `ClassCastException` in `ExceptionConverter extends RuntimeException`. `catch (ClassCastException e)` never triggers — ClassCastException lands in `catch (Exception e)` and returns 100/failed.
- **File handle safety**: `reader.close(); reader = null;` before `FileOutputStream` write — prevents Windows file lock overlap between check API and sign API
- **isRebuilt repair**: Silent — returns `"pdf verification pass"`. Logs use `"rp"` shorthand.
- **ClassCastException / trailer-not-found**: No auto-fix. Returns full exception message. External tools required if fix needed (QPDF / Acrobat Pro).
- **Deep check (2026-07-28)**: `checkPdfError`/`checkPdfErrorFromPath` now also reserve a blank signature placeholder (`PdfStamper.createSignature` → `MakeSignature.signExternalContainer` against an in-memory `ByteArrayOutputStream`) after the `AcroFields` walk. This mirrors `PDF_prepareHash_seal.java`'s real signing path and drives `PdfStamperImp.close()` — the exact frame where the ClassCastException happens — so AcroForm/Annots corruption now surfaces during the check instead of only at production signing time.

## Session History (Last 5)

### 2026-07-28 - Deep signature-placeholder check for ClassCastException false negatives
- **Changes**: Root-caused why `CheckPdfError`/`CheckPdfErrorByPath` returned 000/success for a PDF that later threw `java.lang.ClassCastException: com.itextpdf.text.pdf.PdfArray cannot be cast to com.itextpdf.text.pdf.PdfDictionary` during real signing (traced via `PDF_prepareHash_seal.java:158-159` → `PdfStamperImp.close():217`, from the MyTrustPDFSigner_IT5 signing lab). Existing checker only ran `reader.isRebuilt()` + `AcroFields.getFields()` — neither exercises the code path where the crash actually lives. Added the deep check described above to both SOAP methods. Compiled clean against `itextpdf-5.4.1.jar` + `log4j.jar` via direct `javac` (no `ant` on PATH in this environment).
- **Time Spent**: ~30 min

### 2026-07-03 - Remove PDFBox, ClassCastException returns error
- **Changes**: Removed PDFBox entirely — PDFBox repaired the file but iText5 still triggered ClassCastException on every subsequent call (AcroForm corruption preserved by PDFBox's save). Decision: ClassCastException returns 100/failed with exception message. Removed `isClassCastIssue()` helper, removed PDFBox import, removed pdfbox + fontbox from project.xml and project.properties javac.classpath.
- **Time Spent**: ~15 min

### 2026-07-03 - PDFBox Fallback + ExceptionConverter Fix
- **Changes**: Added PDFBox 2.0.27 as ClassCastException fallback. Discovered iText5 wraps `ClassCastException` in `ExceptionConverter extends RuntimeException`. Added `isClassCastIssue()` helper. Tested — PDFBox repaired structurally but AcroForm corruption was preserved, so 2nd call still triggered repair. Reverted.
- **Time Spent**: ~60 min

### 2026-07-03 - File Handle Safety + Silent Repair
- **Changes**: Fixed `reader.close()` ordering before `FileOutputStream` write. All repair success paths return `"pdf verification pass"`. Log messages use `"rp"` shorthand.
- **Time Spent**: ~15 min

### 2026-07-02 - Project Created
- **Changes**: Scaffolded full project using wildfly-soap-ws skill. Created `CheckPdfError` (Base64) and `CheckPdfErrorByPath` (file path). Added inline auto-repair via `PdfStamper` when `isRebuilt()=true`. Covered all PDF error types. Analyzed repairability: iText only fixes isRebuilt; trailer-not-found requires QPDF/Acrobat.
- **Time Spent**: ~120 min

## Historical Summary
[No history yet — this section is populated when session count exceeds 5]

---
**Last Updated**: 2026-07-28 | **Position**: #1/10 Active
