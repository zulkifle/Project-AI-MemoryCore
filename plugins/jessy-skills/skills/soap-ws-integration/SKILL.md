---
name: soap-ws-integration
description: >
  Use this skill when integrating with SOAP web services in Java — adding a WSDL to Maven,
  using JAX-WS generated stubs, setting HTTP headers via BindingProvider, handling SOAP responses,
  or troubleshooting JAX-WS runtime errors. Trigger when user mentions WSDL, wsimport, SoapUI
  testing, SOAP API integration, or JAX-WS stub usage.
---

# SOAP Web Service Integration — Dejul's Practice

Dejul's standard workflow for integrating with SOAP web services in Java Maven projects.
Always follow this pattern end-to-end.

---

## Workflow Overview

```
SoapUI (test + verify) → pom.xml wsimport → generated stubs → Java implementation
```

1. **SoapUI first** — test the API, verify credentials, check raw HTTP headers
2. **Add WSDL to pom.xml** — generate stubs via `wsimport`
3. **Implement using stubs** — use `BindingProvider` for HTTP headers
4. **Handle response** — use generated result classes directly

---

## Step 1: SoapUI Testing Practice

Before writing any Java code:
- Test the SOAP operation in SoapUI
- Check the **Raw tab** to see exact HTTP request headers being sent
- Verify which headers the server expects (`Username`, `Password`, `Authorization`, etc.)
- Confirm the response structure and field names

> **Why**: SoapUI shows exact working headers. Implement the same in Java.

---

## Step 2: Add WSDL to pom.xml

Follow the existing `jaxws-maven-plugin` execution pattern — add a new `<execution>` block:

```xml
<plugin>
    <groupId>com.sun.xml.ws</groupId>
    <artifactId>jaxws-maven-plugin</artifactId>
    <version>2.3.2</version>
    <executions>

        <!-- Existing WSDLs above... -->

        <!-- New WSDL -->
        <execution>
            <id>[service-name]</id>
            <goals>
                <goal>wsimport</goal>
            </goals>
            <configuration>
                <wsdlUrls>
                    <wsdlUrl>[WSDL_URL]?wsdl</wsdlUrl>
                </wsdlUrls>
                <packageName>com.msctg.[service].prod</packageName>
                <keep>true</keep>
                <sourceDestDir>src/main/java</sourceDestDir>
                <vmArgs>
                    <vmArg>-Djavax.xml.accessExternalSchema=all</vmArg>
                </vmArgs>
            </configuration>
        </execution>

    </executions>
</plugin>
```

Run `mvn generate-sources` to generate stubs into `src/main/java/[packageName]/`.

**Full dependency block** (use this as the standard reference — from TEST-PROJECT):
```xml
<!-- Java EE Web API -->
<dependency>
    <groupId>javax</groupId>
    <artifactId>javaee-web-api</artifactId>
    <version>7.0</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>javax</groupId>
    <artifactId>javaee-api</artifactId>
    <version>8.0</version>
    <type>jar</type>
</dependency>

<!-- JAX-WS — all 4 required together -->
<dependency>
    <groupId>jakarta.xml.ws</groupId>
    <artifactId>jakarta.xml.ws-api</artifactId>
    <version>3.0.0</version>
</dependency>
<dependency>
    <groupId>com.sun.xml.ws</groupId>
    <artifactId>jaxws-rt</artifactId>
    <version>2.3.3</version>  <!-- runtime — prevents ProviderImpl not found -->
</dependency>
<dependency>
    <groupId>javax.xml.ws</groupId>
    <artifactId>jaxws-api</artifactId>
    <version>2.3.1</version>
</dependency>
<dependency>
    <groupId>javax.jws</groupId>
    <artifactId>javax.jws-api</artifactId>
    <version>1.1</version>
</dependency>

<!-- Logging -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-api</artifactId>
    <version>2.18.0</version>
</dependency>
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.18.0</version>
</dependency>
```

> **Key**: All 4 JAX-WS artifacts must be present together. Missing any one causes runtime errors.

---

## Step 3: Java Implementation Pattern

```java
public static String[] callService_pilot(String param1, ...) {
    String[] dec = new String[N];  // size = number of response fields
    try {
        // 1. Create service + port from generated stubs
        MyService_Service service = new MyService_Service();
        MyService port = service.getMyServicePort();

        // 2. Set endpoint URL
        Map<String, Object> req_ctx = ((BindingProvider) port).getRequestContext();
        req_ctx.put(BindingProvider.ENDPOINT_ADDRESS_PROPERTY, WS_ENDPOINT);

        // 3. Set HTTP headers (Username/Password or other custom headers)
        Map<String, List<String>> headers = new HashMap<>();
        headers.put("Username", Collections.singletonList(USERNAME));
        headers.put("Password", Collections.singletonList(PASSWORD));
        req_ctx.put(MessageContext.HTTP_REQUEST_HEADERS, headers);

        // 4. Call the operation
        MyResultClass result = port.myOperation(param1, ...);

        // 5. Map response fields — each field to its own index
        dec[0] = result.getStatusCode();
        dec[1] = result.getStatusMsg();
        dec[2] = result.getFieldA();
        dec[3] = result.getFieldB();

    } catch (Exception e) {
        logger.error("callService_pilot exception: {}", e.getMessage(), e);
        return null;
    }
    return dec;
}
```

**Key rules:**
- Array size = exact number of fields returned (never reuse the same index)
- Always check `result == null` after calling the pilot method
- Check `statusCode` before using other fields
- Log the `statusCode` after successful call

---

## Step 4: Response Handling

```java
String[] result = callService_pilot(...);

if (result == null) {
    throw new Exception("Service call failed — null response");
}

String statusCode = result[0];
String statusMsg  = result[1];

// Handle known error codes
if ("DS999".equals(statusCode)) {
    throw new SessionExpiredException("Session expired: " + statusMsg);
}
if (!"000".equals(statusCode)) {
    throw new Exception("Service failed [" + statusCode + "]: " + statusMsg);
}

// Use result fields
String data = result[2];
```

---

## Common Issues & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `ProviderImpl not found` | JAX-WS API present but no runtime | Add `com.sun.xml.ws:jaxws-rt:2.3.2` to pom.xml |
| `Invalid API credential` | Wrong header name or format | Check SoapUI Raw tab for exact header names |
| `MessageDigest (provider: BC) cannot be found` | BouncyCastle not registered as JCE provider | Copy `bcprov-*.jar` to JVM ext + add to `java.security` in Dockerfile |
| `WS102: Invalid credential` | Credentials correct but sent wrong way | Verify HTTP header names match exactly (case-sensitive) |
| `WS111: Username missing` | Custom HTTP headers not being sent | Use `MessageContext.HTTP_REQUEST_HEADERS`, not `setRequestProperty` |

---

## BouncyCastle JCE Fix (Docker)

When deploying in Docker and MTSA/signing service needs BC:

```dockerfile
# Register BouncyCastle as JCE security provider
RUN JAVA_SEC=$(find /usr/lib/jvm -name "java.security" -path "*/jre/*" | head -1) && \
    JAVA_EXT=$(dirname "$JAVA_SEC")/../ext && \
    cp /path/to/bcprov-*.jar "$JAVA_EXT/" && \
    LAST=$(grep -E "^security\.provider\.[0-9]+" "$JAVA_SEC" | tail -1 | sed 's/security\.provider\.\([0-9]*\)=.*/\1/') && \
    echo "security.provider.$((LAST + 1))=org.bouncycastle.jce.provider.BouncyCastleProvider" >> "$JAVA_SEC"
```

---

## Log4j2 Standard Setup

### 1. Config file — `src/main/resources/log4j2.properties`

```properties
property.basePath = /opt/lhdn/logs/
#property.basePath = D:\\opt\\lhdn\\logs\\   <- uncomment for local Windows dev

appender.console.type = Console
appender.console.name = STDOUT
appender.console.layout.type = PatternLayout
appender.console.layout.pattern = [%-5level] %d{yyyy-MM-dd HH:mm:ss.SSS} (%F:%L) - %msg%n

appender.rolling.type = RollingFile
appender.rolling.name = RollingFile
appender.rolling.fileName = ${basePath}einvoice.log
appender.rolling.filePattern = ${basePath}einvoice_%d{ddMMyy}.log
appender.rolling.layout.type = PatternLayout
appender.rolling.layout.pattern = %d{yyyy-MM-dd HH:mm:ss.SSS} %level (%F:%L) - %msg%n
appender.rolling.policies.type = Policies
appender.rolling.policies.time.type = TimeBasedTriggeringPolicy
appender.rolling.policies.time.interval = 1
appender.rolling.policies.time.modulate = true
appender.rolling.strategy.type = DefaultRolloverStrategy
appender.rolling.strategy.max = 5

appender.rolling.strategy.delete.type = Delete
appender.rolling.strategy.delete.basePath = ${basePath}
appender.rolling.strategy.delete.maxDepth = 10
appender.rolling.strategy.delete.ifLastModified.type = IfLastModified

# Delete all files older than 180 days
appender.rolling.strategy.delete.ifLastModified.age = 180d

rootLogger.level = INFO
rootLogger.appenderRef.stdout.ref = STDOUT
rootLogger.appenderRef.RollingFile.ref = RollingFile
```

> Change `einvoice` to your project name. Log path `/opt/lhdn/logs/` for Linux/Docker, comment in Windows path for local dev.

---

### 2. Logger declaration in every class

```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class MyClass {
    private static final Logger logger = LogManager.getLogger(MyClass.class);
}
```

### 3. Usage patterns

```java
logger.info("callService — param: {}", param);          // method entry
logger.info("callService success — statusCode: {}", code); // success
logger.warn("callService DS999 — session expired for: {}", userId); // known warning
logger.error("callService exception: {}", e.getMessage(), e);       // catch block
```

---

## Required Imports

**Jakarta EE 9+ / Spring Boot 3 (use `jakarta.*`):**
```java
import jakarta.xml.ws.BindingProvider;
import jakarta.xml.ws.handler.MessageContext;
import java.util.List;
import java.util.Map;
```

**Java EE 8 / Spring Boot 2 and below (use `javax.*`):**
```java
import javax.xml.ws.BindingProvider;
import javax.xml.ws.handler.MessageContext;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
```

> **Rule**: Match imports to the Jakarta namespace of your project. Spring Boot 3 uses `jakarta.*`. Never mix `javax.*` and `jakarta.*`.

---

## Spring Boot 3 — JAX-WS as a `@Service` (NOT static methods)

When integrating SOAP in a Spring Boot 3 project, use the `@Service` + `@RequiredArgsConstructor` pattern.
**Never use static utility methods** — inject config via `@ConfigurationProperties` instead.

### pom.xml (Spring Boot 3 / Jakarta EE 9+)

```xml
<!-- JAX-WS runtime for Spring Boot 3 -->
<dependency>
    <groupId>com.sun.xml.ws</groupId>
    <artifactId>jaxws-rt</artifactId>
    <version>4.0.1</version>
</dependency>

<!-- wsimport plugin -->
<plugin>
    <groupId>com.sun.xml.ws</groupId>
    <artifactId>jaxws-maven-plugin</artifactId>
    <version>4.0.0</version>
    <executions>
        <execution>
            <id>[service-name]</id>
            <goals><goal>wsimport</goal></goals>
            <configuration>
                <wsdlUrls>
                    <wsdlUrl>[WSDL_URL]?wsdl</wsdlUrl>
                </wsdlUrls>
                <packageName>com.example.generated</packageName>
                <keep>true</keep>
                <sourceDestDir>src/main/java</sourceDestDir>
                <vmArgs>
                    <vmArg>-Djavax.xml.accessExternalSchema=all</vmArg>
                </vmArgs>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Service Implementation Pattern (Spring Boot 3)

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class MtssServiceImpl implements MtssService {

    private final MtssProperties mtssProperties;

    @Override
    public void callCheckEkyc(String userId, String ekycStatus, String fullname,
                               String frontImg, String backImg, String selfieImg,
                               String fullResponse) {

        // 1. Create service + port from generated stubs
        GetResultJumio_Service soapService = new GetResultJumio_Service();
        GetResultJumio port = soapService.getGetResultJumioPort();

        // 2. Override endpoint URL + set HTTP headers via BindingProvider
        BindingProvider bp = (BindingProvider) port;
        bp.getRequestContext().put(BindingProvider.ENDPOINT_ADDRESS_PROPERTY, mtssProperties.getEndpointUrl());
        bp.getRequestContext().put(MessageContext.HTTP_REQUEST_HEADERS, Map.of(
                "Username",   List.of(mtssProperties.getUsername()),
                "Password",   List.of(mtssProperties.getPassword()),
                "ProjectKey", List.of(mtssProperties.getProjectKey())
        ));

        // 3. Call the operation — clean, no XML building
        try {
            ReturnData result = port.checkEKYCresultJumio(
                    userId, ekycStatus, fullname, frontImg, backImg, selfieImg, fullResponse);
            log.info("MTSS statusCode={}, statusMsg={}", result.getStatusCode(), result.getStatusMsg());
        } catch (Exception e) {
            log.error("MTSS SOAP call failed: {}", e.getMessage(), e);
            throw new RuntimeException("MTSS SOAP call failed", e);
        }
    }
}
```

> **NEVER use `buildSoapEnvelope` + `RestTemplate` for SOAP.** Always use the JAX-WS generated stubs.
> This was a mistake found in jumio-proxy-integration (2026-04-17) — stubs were generated but not used.

---

## Java Version Compatibility

This full setup (JAX-WS + Log4j2) works on **JDK 8 and above** — confirmed compatible with JDK 8, 11, 17.

When starting a project, set compiler source/target in pom.xml to match your JDK:

```xml
<properties>
    <maven.compiler.source>1.8</maven.compiler.source>  <!-- JDK 8 -->
    <maven.compiler.target>1.8</maven.compiler.target>
    <!-- Use 11 or 17 for newer JDKs -->
</properties>
```

Also align the `maven-compiler-plugin` explicitly — properties alone can be overridden:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.1</version>
    <configuration>
        <source>1.8</source>   <!-- match your JDK -->
        <target>1.8</target>
    </configuration>
</plugin>
```

> Note: TEST-PROJECT has a conflict — properties say `11` but plugin config says `1.7`. Plugin config wins. Always set both consistently.

---

## Reference Projects

| Project | Path | Notes |
|---|---|---|
| DSPortalDemo | `C:\PROJECTS\DEMO\MYEG-SIGNING-PORTAL\Development\DSPortalDemo\` | SignPDFFile + GetCertInfo — single WSDL (mtsaws) |
| TEST-PROJECT | `C:\PROJECTS\DEMO\TEST-PROJECT\` | 3 WSDLs: pkiws, roamingws, mtsaws — use as dependency template |

---

## Level History
- **Lv.1** — Base: SoapUI-first workflow, pom.xml wsimport pattern, BindingProvider HTTP headers,
  response mapping, common error fixes. Derived from signingDemoPortal session (2026-04-06).
- **Lv.2** — Added full dependency block (all 4 JAX-WS artifacts + Log4j). Added reference projects table.
  Added Log4j2 standard setup (log4j2.properties template, logger declaration, usage patterns).
  Source: TEST-PROJECT (2026-04-10).
- **Lv.3** — Added Spring Boot 3 / Jakarta EE 9+ section: `jaxws-rt:4.0.1`, `jaxws-maven-plugin:4.0.0`,
  `jakarta.*` imports, `@Service` pattern with `@RequiredArgsConstructor`. Added explicit rule:
  NEVER use `buildSoapEnvelope` + RestTemplate — always use JAX-WS stubs.
  Source: jumio-proxy-integration (2026-04-17).
