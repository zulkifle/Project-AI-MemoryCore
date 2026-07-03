# ECOURT_PdfErrorCheckWS - PDF Error Check SOAP Web Service
*JAX-WS SOAP service on WildFly to check and auto-repair PDFs before digital signing*

## Project Overview
- **Type**: API (SOAP Web Service)
- **Client**: Internal (POJ / e-Court)
- **Period**: 2026-07-02 - Active
- **Tech Stack**: Java 8 + JAX-WS + WildFly 18 + iText5 5.4.1 + PDFBox 2.0.27 + log4j
- **Completion**: 95%
- **Duration**: ~3.5 hours
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-07-03 — PDFBox fallback added, ExceptionConverter bug found and fixed, tested OK
- **Next Steps**:
  1. Deploy to production server
  2. Monitor logs for any new PDF error types
- **Known Issues**:
  - `InvalidPdfException: Rebuild failed: trailer not found` → PDF too corrupt for iText + PDFBox, external repair required (QPDF / Acrobat Pro)

## SOAP Methods

| Method | Params | Purpose |
|--------|--------|---------|
| `CheckPdfError` | `PdfBase64`, `FileName` | Check PDF from Base64; auto-rp if needed, in memory |
| `CheckPdfErrorByPath` | `PDFtoSign`, `gUid` | Check PDF from file path; auto-rp if needed, overwrite file |

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
| `isRebuilt() = true` (minor xref damage) | Yes — PdfStamper | 000 / success / pdf verification pass |
| `ClassCastException: PdfArray/PdfNumber cast` | Yes — PdfStamper → PDFBox fallback | 000 / success / pdf verification pass |
| `InvalidPdfException: Rebuild failed` | No — too corrupt | 100 / failed / full exception message |

## Repair Chain (internal — client never sees)
```
isRebuilt() = true
  → PdfStamper → overwrite file → 000/success

ClassCastException (wrapped as ExceptionConverter by iText5)
  → detected via isClassCastIssue() helper (instanceof + getCause() + toString())
  → Try 1: PdfStamper → 000/success
  → Try 2: PDFBox → 000/success
  → Both fail → 100/failed
```

## Project Structure
```
ECOURT_PdfErrorCheckWS/
├── src/java/com/poj/pdferrorcheck/
│   ├── PdfErrorCheckService.java   ← @WebService, 2 methods
│   ├── PdfErrorCheckHelper.java    ← logic, auto-rp inline, isClassCastIssue()
│   └── PdfCheckResult.java         ← errCode, status, errMsg
├── web/WEB-INF/
│   ├── jboss-web.xml               ← context-root: /PdfErrorCheckWS
│   └── web.xml                     ← minimal (WildFly handles @WebService)
├── nbproject/
│   ├── project.xml                 ← libs: log4j + itextpdf-5.4.1 + pdfbox + fontbox
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
- **Key Dependencies**: `itextpdf-5.4.1.jar`, `pdfbox-2.0.27.jar`, `fontbox-2.0.27.jar`, `log4j.jar` — all in `C:\PROJECTS\java Library\`
- **iText5 ExceptionConverter**: iText5 wraps `ClassCastException` in `ExceptionConverter extends RuntimeException` — `catch (ClassCastException e)` never triggers. Must detect via `isClassCastIssue()` in `catch (Exception e)`.
- **File handle safety**: `reader.close(); reader = null;` before `FileOutputStream` write — prevents Windows file lock overlap between check API and sign API
- **Repair is silent**: All repair success paths return `"pdf verification pass"` — client never knows repair happened. Internal logs use `"rp"` shorthand.
- **For trailer-not-found**: External tools only — QPDF (`qpdf --recover`) or Acrobat Pro Method 3 (Reduced Size PDF)

## Session History (Last 5)

### 2026-07-03 - PDFBox Fallback + ExceptionConverter Fix
- **Changes**: Added PDFBox 2.0.27 as ClassCastException fallback (pdfbox + fontbox jars added to project.xml + project.properties). Discovered root cause: iText5 wraps `ClassCastException` in `ExceptionConverter extends RuntimeException`, so `catch (ClassCastException e)` was never triggered — ClassCastException was silently going to `catch (Exception e)`. Fixed by moving detection to `isClassCastIssue()` helper method inside `catch (Exception e)`. Repair chain: PdfStamper first, PDFBox if PdfStamper fails. All repair success paths return `"pdf verification pass"` — client unaware of repair. Log messages use `"rp"` instead of `"repair"`. Tested and confirmed working.
- **Time Spent**: ~60 min

### 2026-07-03 - File Handle Safety Fix
- **Changes**: Fixed `reader.close()` ordering in both repair blocks. Reader closed before `FileOutputStream` write to prevent Windows file lock overlap when client calls check API then sign API sequentially.
- **Time Spent**: ~15 min

### 2026-07-02 - Project Created
- **Changes**: Scaffolded full project using wildfly-soap-ws skill. Created `CheckPdfError` (Base64) and `CheckPdfErrorByPath` (file path). Added inline auto-repair via `PdfStamper` when `isRebuilt()=true`. Covered all PDF error types. Analyzed repairability: iText only fixes isRebuilt; trailer-not-found requires QPDF/Acrobat.
- **Time Spent**: ~120 min

## Historical Summary
[No history yet — this section is populated when session count exceeds 5]

---
**Last Updated**: 2026-07-03 | **Position**: #1/10 Active
