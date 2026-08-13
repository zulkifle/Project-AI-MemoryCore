# ECOURT_PdfErrorCheckWS - PDF Error Check SOAP Web Service
*JAX-WS SOAP service on WildFly to check and auto-repair PDFs before digital signing*

## Project Overview
- **Type**: API (SOAP Web Service)
- **Client**: Internal (POJ / e-Court)
- **Period**: 2026-07-02 - Active
- **Tech Stack**: Java 8 + JAX-WS + WildFly 18 + iText5 5.4.1 + log4j
- **Completion**: 99%
- **Duration**: ~5.5 hours
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-08-13 — Added new Stream API (`/PdfErrorCheckStream` servlet) for byte-array PDF checking — avoids Base64 network overhead. Client integration doc + Postman/.NET/Java/curl samples written, ready to share.
- **Previous Session**: 2026-08-06 — Dejul confirmed the deep-check fix is tested OK and live on the client's pilot server; false-negative resolved in real usage
- **Next Steps**:
  1. Zul: deploy updated WAR to dev/pilot WildFly, run `testCheckStream.java` (or Postman) against a known-good and known-corrupt PDF to confirm parity with the SOAP method before sharing the doc with the client
  2. Decide on log retention: add the PowerShell cleanup script + Scheduled Task (10-day retention) for `PdfErrorCheck.log.*`, or leave logs unmanaged and close the project as-is — awaiting Zul's call
- **Known Issues**:
  - `ClassCastException: PdfArray cannot be cast to PdfDictionary` → 100/failed, no auto-fix (AcroForm/Annots structure corruption — not fixable without external tools). Previously this could slip through as 000/success on some PDFs because the check didn't reach the code path where it occurs — see 2026-07-28 session.
  - `InvalidPdfException: Rebuild failed: trailer not found` → 100/failed, too corrupt for iText

## SOAP Methods

| Method | Params | Purpose |
|--------|--------|---------|
| `CheckPdfError` | `PdfBase64`, `FileName` | Check PDF from Base64; auto-rp xref damage in memory |
| `CheckPdfErrorByPath` | `PDFtoSign`, `gUid` | Check PDF from file path; auto-rp xref damage, overwrite file |

## Stream API (Servlet, added 2026-08-13)

| Endpoint | Input | Purpose |
|----------|-------|---------|
| `POST /PdfErrorCheckWS/PdfErrorCheckStream` | Raw PDF bytes (body, `application/octet-stream`) + `X-File-Name` header | Same in-memory check as `CheckPdfError`, but skips Base64 — avoids ~33% payload inflation over the network |

Not a SOAP operation — plain `HttpServlet`, same web app/context root. Response is JSON
(not XML), same errCode/status/errMsg values. Full client integration guide (curl, Java,
.NET, Postman): `docs/pdf-error-check-stream-api.md`.

## Response Format

SOAP methods return XML:
```xml
<errCode>000</errCode>   <!-- 000=success, 100=failed -->
<status>success</status> <!-- success / failed -->
<errMsg>pdf verification pass</errMsg>
```

Stream API returns the same values as JSON:
```json
{"errCode": "000", "status": "success", "errMsg": "pdf verification pass"}
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
│   ├── PdfErrorCheckService.java       ← @WebService, 2 SOAP methods
│   ├── PdfErrorCheckHelper.java        ← logic: isRebuilt auto-rp + deep signature-placeholder check
│   ├── PdfErrorCheckStreamServlet.java ← new: byte-array stream servlet, JSON response
│   ├── PdfCheckResult.java             ← errCode, status, errMsg
│   └── test/testCheckStream.java       ← new: HttpURLConnection sample/local test client
├── web/WEB-INF/
│   ├── jboss-web.xml               ← context-root: /PdfErrorCheckWS
│   └── web.xml                     ← minimal (WildFly handles @WebService + @WebServlet)
├── nbproject/
│   ├── project.xml                 ← libs: log4j + itextpdf-5.4.1 + json-20230618
│   ├── project.properties          ← JDK8, WildFly, war.name=PdfErrorCheckWS.war
│   └── jax-ws.xml                  ← registers PdfErrorCheckService
├── docs/
│   └── pdf-error-check-stream-api.md ← client-facing integration guide (curl/Java/.NET/Postman)
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
- **No document storage, either API**: `CheckPdfError` (Base64) processes entirely in memory (`ByteArrayOutputStream` buffers) — no `File`/`FileOutputStream` calls in that path at all. `CheckPdfErrorByPath` only reads a file the *caller* already placed on disk and, on repair, overwrites that same file in place — it never creates a second stored copy. Confirmed by reading `PdfErrorCheckHelper.java` end to end (2026-08-06) in response to a client question about storage capacity at 100K+ docs/day — answer: not applicable, nothing is stored.
- **Logging caveat**: `log4j.properties` uses `DailyRollingFileAppender` (date-based rolling, one file per day at `C:\Trustgate\MyTrustSignServer\logs\PdfErrorCheck.log`). `MaxBackupIndex` is a `RollingFileAppender`-only property (size-based rolling) — log4j 1.x silently ignores it on `DailyRollingFileAppender`, so it does **not** cap retention despite looking like it should. Log content is filenames/gUid/status/exception text only, never PDF bytes — so this is a housekeeping concern, not a data-sensitivity one. Real fix for bounded retention: an external scheduled cleanup (PowerShell + Windows Scheduled Task deleting files older than N days), not a log4j property.
- **Stream API design (2026-08-13)**: `PdfErrorCheckHelper` refactored — extracted shared `checkPdfErrorBytes(byte[])` private method from `checkPdfError`'s post-Base64-decode logic. New `checkPdfErrorFromStream(byte[], fileName)` calls it directly, so the Stream API's checking logic is byte-for-byte identical to `CheckPdfError` (same `isRebuilt` auto-repair → AcroForm walk → blank signature placeholder), zero drift risk between the two entry points. `checkPdfErrorFromPath` (file-overwrite variant) left untouched — different behavior contract, not in scope.
- **Stream API transport decisions** (confirmed with Zul): JSON response (not XML) since it's not a SOAP operation; `fileName` passed via `X-File-Name` HTTP header (not query string) since the body is 100% raw bytes; `application/octet-stream` body (not `multipart/form-data`) since there's only one payload and no other form fields to carry — avoids multipart boundary parsing overhead entirely.
- **JSON library choice**: Used `org.json.JSONObject` (`json-20230618.jar`, added to `project.properties`/`project.xml`) instead of `javax.json` — matches the existing convention already used elsewhere in the POJ codebase (`RoamingWSV3_POJ_Wildfly_LOCALSIGN/pkiws.java`).
- **Compile verification**: No `ant` on PATH (same as 2026-07-28 session) — compiled `PdfErrorCheckHelper.java` + `PdfErrorCheckStreamServlet.java` + `testCheckStream.java` directly via `javac` against `log4j.jar` + `itextpdf-5.4.1.jar` + `json-20230618.jar` + WildFly's bundled `jboss-servlet-api_4.0_spec-2.0.0.Final.jar` (found at `C:\wildfly-18.0.1.Final\modules\system\layers\base\javax\servlet\api\main\`). Clean compile, zero errors. Not yet deployed/live-tested — WAR redeploy + Postman/`testCheckStream` run still pending (Zul's next step).

## Session History (Last 5)

### 2026-08-13 - Stream API added (byte-array, no Base64)
- **Changes**: Added a new byte-array Stream API to cut network payload size vs. Base64. Built as a plain servlet (`PdfErrorCheckStreamServlet`, `POST /PdfErrorCheckWS/PdfErrorCheckStream`), not SOAP — takes raw PDF bytes in the request body (`application/octet-stream`) + `X-File-Name` header, returns JSON. Refactored `PdfErrorCheckHelper` to share the exact same checking logic as `CheckPdfError` via a new private `checkPdfErrorBytes()` — zero behavior drift between SOAP and stream entry points. Added `org.json` (`json-20230618.jar`) for the JSON response, matching existing POJ codebase convention. Wrote a client integration doc (`docs/pdf-error-check-stream-api.md`) covering curl, Java, .NET (`HttpClient`), and Postman, plus a local test client (`testCheckStream.java`) matching the existing `testCheckBase64.java` convention. Compiled clean via direct `javac` against WildFly's bundled servlet-api jar. Not yet deployed/live-tested.
- **Time Spent**: ~45 min

### 2026-08-13 - Project resumed
- **Changes**: Project resumed from position #2, moved to position #1.
- **Time Spent**: —

### 2026-07-29 - Redeployed to client's pilot environment
- **Changes**: Redeployed the WAR with the 2026-07-28 deep signature-placeholder check to the client's pilot environment.
- **Time Spent**: —

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
**Last Updated**: 2026-08-13 (Stream API added) | **Position**: #1/10 Active
