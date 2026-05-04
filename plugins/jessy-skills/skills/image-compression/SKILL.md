# image-compression — Scalr-Based Image Compression Skill
*Compress base64 images before sending to SOAP/external services*

## Activation
Trigger when user says "include image compression", "add image compression", or mentions compressing
images in any Java project. Provides `ImageCompressionUtil.java` + `Scalr.java` as reusable utility classes
to be added to the project — not tied to any specific use case.

## Origin
Derived from production code used in MSC Trustgate project (2024-02-28).
Files studied: `compressFunction.java` + `Scalr.java`

---

## Core Pattern

### Input/Output Contract
```java
// Returns String[3]: {errcode, errmsg, compressedBase64}
// "000" = success, "CR113" = error
// If below threshold → returns original base64 unchanged
private String[] compressImage(String base64Input, ...)
```

### Decision Flow
```
base64 input
    ↓
computeDecodedByteSize()  ← check size FIRST
    ↓
> MAX_COMPRESSION_SIZE?
    YES → Scalr.resize() → write as JPEG → re-encode base64 → return compressed
    NO  → return original as-is
```

### Scalr Usage
```java
BufferedImage img = Scalr.resize(ImageIO.read(input), cfg.targetSize);
// targetSize = max width/height in pixels (proportional, no distortion)
// Uses bicubic interpolation — high quality, no blur artifacts
```

### ImageIO Writer (force JPEG output)
```java
Iterator<ImageWriter> writers = ImageIO.getImageWritersByFormatName("jpg");
ImageWriter writer = writers.next();
ImageOutputStream ios = ImageIO.createImageOutputStream(os);
writer.setOutput(ios);
ImageWriteParam param = writer.getDefaultWriteParam();
// Optional: param.setCompressionMode(ImageWriteParam.MODE_EXPLICIT);
//           param.setCompressionQuality(0.85f);  // 0.0=worst, 1.0=best
writer.write(null, new IIOImage(img, null, null), param);
```

---

## In-Memory Adaptation (for microservices — no temp files)

Use `ByteArrayOutputStream` instead of writing to disk:

```java
public String compressToBase64(String base64Input, int maxBytes, int targetSize) throws Exception {
    byte[] imageBytes = Base64.getDecoder().decode(base64Input);
    if (imageBytes.length <= maxBytes) return base64Input; // skip if under threshold

    BufferedImage img = ImageIO.read(new ByteArrayInputStream(imageBytes));
    BufferedImage resized = Scalr.resize(img, targetSize);

    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    ImageOutputStream ios = ImageIO.createImageOutputStream(baos);

    Iterator<ImageWriter> writers = ImageIO.getImageWritersByFormatName("jpg");
    ImageWriter writer = writers.next();
    writer.setOutput(ios);

    ImageWriteParam param = writer.getDefaultWriteParam();
    writer.write(null, new IIOImage(resized, null, null), param);

    writer.dispose();
    ios.close();

    return Base64.getEncoder().encodeToString(baos.toByteArray());
}
```

---

## Format Detection from Base64

```java
public String detectImageFormat(String base64) {
    if (base64.startsWith("/9j/"))      return "jpg";
    if (base64.startsWith("iVBORw0K")) return "png";
    if (base64.startsWith("R0lGOD"))   return "gif";
    return "unknown";
}
```

- JPEG: `/9j/` prefix
- PNG: `iVBORw0K` prefix
- If PNG detected, still output as JPEG (lossy but smaller) — acceptable for eKYC face images

---

## Config Values (reference from original)

| Config Key | Meaning | Typical Value |
|---|---|---|
| `MAX_COMPRESSION_SIZE` | Threshold in decoded bytes — skip if below | `1_000_000` (1MB) |
| `targetSize` | Max pixel dimension for Scalr resize | `800` or `1024` |
| `compressionQuality` | JPEG quality (if MODE_EXPLICIT used) | `0.85f` |

---

## Jumio Proxy Integration Point

**Where to plug in** — `JumioCallbackController.java`, after `getImageAsBase64()` and before `callCallbackJumio()`:

```
download image (href) → getImageAsBase64()
    ↓
detect format
    ↓
compressToBase64() if needed
    ↓
callCallbackJumio(compressedBase64, ...)
```

**Class to create**: `util/ImageCompressionUtil.java`
- Static utility, no Spring bean needed
- Copy `Scalr.java` into `util/Scalr.java` (already exists as standalone class)

---

## Required Imports
```java
import java.awt.image.BufferedImage;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.util.Base64;
import java.util.Iterator;
import javax.imageio.IIOImage;
import javax.imageio.ImageIO;
import javax.imageio.ImageWriteParam;
import javax.imageio.ImageWriter;
import javax.imageio.stream.ImageOutputStream;
```

Note: Uses `javax.imageio` (standard Java SE) — NOT `jakarta.*`. Safe for Spring Boot 3.

---

## Error Handling
- Wrap in try-catch, return original base64 on failure (fail-safe — don't block SOAP call)
- Log: image format, size before/after compression
- Error code convention from original: `CR113` = compression error

---

## Level History
- **Lv.1** — Base: Pattern extracted from production compressFunction.java + Scalr.java.
  In-memory adaptation added. Format detection added. Jumio proxy integration point documented.
