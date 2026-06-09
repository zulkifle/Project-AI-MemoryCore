---
name: mtsa-container-packaging
description: "MUST use when user says 'package [project]', 'create deployment package',
             'create docker package', 'dockerize MTSA',
             or when user needs to package an MTSA deployment folder into a Docker-ready ZIP
             (Dockerfile + mtsa/ + webapps/ + STEP.txt + .zip). Also triggers on:
             'container packaging', 'MTSA package', 'deployment ZIP', 'prepare for handover'."
---

# MTSA Container Packaging — Docker Deployment Package Creator
*Packages an MTSA deployment folder into a Docker-ready ZIP: Dockerfile + STEP.txt + folders + .zip*

## Activation

When this skill activates, output:

`📦 MTSA Container Packaging — ready. Gathering project info...`

Then execute the protocol below.

## Context Guard

| Context | Status |
|---------|--------|
| **User wants to package an MTSA deployment folder** | ACTIVE — full protocol |
| **User mentions Dockerfile + mtsa folder + zip** | ACTIVE — full protocol |
| **General Docker question unrelated to MTSA** | DORMANT — do not activate |
| **Spring Boot / Laravel packaging** | DORMANT — different skills handle those |

---

## Protocol

### Step 1: Gather Project Info

Display this intake form and wait for user to fill in all required fields:

```
╔══════════════════════════════════════════════════════════╗
║         MTSA CONTAINER PACKAGING — PROJECT INFO          ║
╚══════════════════════════════════════════════════════════╝

  Client Name*        : (e.g., AMANAH KREDIT)
  Environment*        : [ ] PILOT   [ ] PRODUCTION
  Deployment Path*    : (e.g., C:\PROJECTS\ELENDING\AMANAH KREDIT\Deployment\PILOT)
  Docker Image Name*  : (e.g., mtsa-pilot:1.01)
  Container Name*     : (e.g., mtsapilot-container-amanah)
  Port Mapping        : (default: 8080:8080)
  WAR Context Path*   : (e.g., MTSAPilot)
  WSDL Endpoint*      : (e.g., MyTrustSignerAgentWSAPv2?wsdl)
  WSDL Host URL*      : (e.g., http://localhost:8080)

══════════════════════════════════════════════════════════
  Fill in the fields above and I'll generate the package.
══════════════════════════════════════════════════════════
```

Parse from user answers:
- `[CLIENT]` — client name (hyphenated for ZIP name, e.g., AMANAH-KREDIT)
- `[ENV]` — PILOT or PRODUCTION
- `[DEPLOY_PATH]` — full path to the deployment folder
- `[IMAGE]` — Docker image name:tag
- `[CONTAINER]` — Docker container name
- `[PORT]` — host:container port (default `8080:8080`)
- `[WAR_CONTEXT]` — WAR context path (e.g., `MTSAPilot`)
- `[WSDL_ENDPOINT]` — WSDL path (e.g., `MyTrustSignerAgentWSAPv2?wsdl`)
- `[WSDL_HOST]` — base URL (e.g., `http://localhost:8080`)

---

### Step 2: Verify Folder Structure

Before creating any file, verify the deployment folder has the required structure:

```powershell
ls "[DEPLOY_PATH]"
ls "[DEPLOY_PATH]\mtsa"
ls "[DEPLOY_PATH]\webapps"
```

- [ ] Confirm `mtsa\` folder exists — contains `.properties`, `.xml`, config files
- [ ] Confirm `mtsa\` has subfolders: `files\`, `logs\`, `tmp\` — create them if missing
- [ ] Confirm `webapps\` folder exists — contains the `.war` file and Tomcat apps
- [ ] If `mtsa\` or `webapps\` root is missing: **STOP** and inform user. Do not proceed.
- [ ] Report what was found in each folder before continuing

---

### Step 3: Create Dockerfile

Write the Dockerfile to `[DEPLOY_PATH]\Dockerfile`:

```dockerfile
FROM tomcat:9-jdk17-corretto

USER root

COPY webapps /usr/local/tomcat/webapps/
COPY mtsa /opt/mtsa

EXPOSE 8080

ENV TZ=Asia/Kuala_Lumpur
RUN yum update -y && \
    yum install -y tzdata && \
    ln -fs /usr/share/zoneinfo/$TZ /etc/localtime && \
    echo $TZ > /etc/timezone && \
    yum clean all && \
    rm -rf /var/cache/yum/*

CMD ["sh", "-c", "/usr/local/tomcat/bin/catalina.sh run"]
```

- [ ] If `Dockerfile` already exists: show diff and ask user to confirm overwrite before proceeding
- [ ] Write the file after confirmation (or immediately if no existing file)

---

### Step 4: Create STEP.txt

Write `[DEPLOY_PATH]\STEP.txt` with values filled in from Step 1:

```
step-by-step instructions

1. Set JAVA_HOME to JDK 17

2. Build the Docker Image
   > docker build -t [IMAGE] .

3. Run the Container
   > docker run -d -p [PORT] --name [CONTAINER] --restart=always [IMAGE]

4. Verify Tomcat is Running
   > docker logs [CONTAINER]

5. Access via Web Browser
   > [WSDL_HOST]/[WAR_CONTEXT]/[WSDL_ENDPOINT]

6. Done. Ready to test!
```

- [ ] If `STEP.txt` already exists: show current content, ask user to confirm overwrite
- [ ] Substitute all placeholders with actual values from Step 1
- [ ] Write the file after confirmation (or immediately if no existing file)

---

### Step 5: Create ZIP Package

Generate the ZIP file using PowerShell `Compress-Archive`.

ZIP file name pattern: `MTSA-[ENV]_[CLIENT-HYPHENATED].zip`
- Example: `MTSA-PILOT_AMANAH-KREDIT.zip`

ZIP destination: one level up from the deployment folder (sibling to the env folder), or same folder if root is already deployment root.

```powershell
$deployPath = "[DEPLOY_PATH]"
$zipName    = "MTSA-[ENV]_[CLIENT-HYPHENATED].zip"
$zipDest    = Join-Path (Split-Path $deployPath -Parent) $zipName

# Remove old ZIP if exists
if (Test-Path $zipDest) { Remove-Item $zipDest -Force }

# Zip only required items
Compress-Archive -Path "$deployPath\Dockerfile",
                       "$deployPath\mtsa",
                       "$deployPath\webapps",
                       "$deployPath\STEP.txt" `
                 -DestinationPath $zipDest

Write-Host "ZIP created: $zipDest"
```

- [ ] Run the PowerShell command
- [ ] Confirm the ZIP file was created and report its size and path
- [ ] If an existing ZIP already exists: remove it first (handled by `-Force` logic above)

---

### Step 6: Confirm & Summary

Display a completion summary:

```
╔══════════════════════════════════════════════════════════╗
║           MTSA PACKAGE — COMPLETE ✓                      ║
╚══════════════════════════════════════════════════════════╝

  Client      : [CLIENT]
  Environment : [ENV]
  Folder      : [DEPLOY_PATH]

  Files Created:
  ✓ Dockerfile
  ✓ STEP.txt
  ✓ [ZIP_FILE_NAME] → [ZIP_DEST_PATH]

  ZIP Contains:
  • Dockerfile
  • mtsa\       (config + properties)
  • webapps\    (WAR + Tomcat apps)
  • STEP.txt    (deployment guide)

  Next Steps:
  1. Transfer ZIP to the target server
  2. Extract into deployment folder
  3. Follow STEP.txt to build and run the container
══════════════════════════════════════════════════════════
```

---

## Mandatory Rules

1. **Always verify folders first** — never create files if `mtsa\` or `webapps\` are missing
2. **Ask before overwriting** — if `Dockerfile` or `STEP.txt` already exist, show diff and confirm
3. **URLs must be filled in** — never leave placeholder text like `[WSDL_HOST]` in STEP.txt; always substitute real values
4. **ZIP must include exactly 4 items** — `Dockerfile`, `mtsa\`, `webapps\`, `STEP.txt`; nothing else unless user explicitly asks
5. **Dockerfile is standard** — do not modify the Dockerfile template unless user specifically requests a change
6. **ZIP naming follows convention** — `MTSA-[ENV]_[CLIENT-HYPHENATED].zip` — uppercase ENV, hyphenated client name
7. **Remove old ZIP before re-zipping** — never append to an existing ZIP; always recreate fresh

---

## Edge Cases

| Situation | Behavior |
|-----------|----------|
| `mtsa\` folder missing | STOP — inform user, list what was found, do not create any files |
| `webapps\` folder missing | STOP — inform user, list what was found, do not create any files |
| Dockerfile already exists | Show current content, ask to confirm overwrite before writing |
| STEP.txt already exists | Show current content, ask to confirm overwrite before writing |
| ZIP already exists at destination | Remove old ZIP first, then recreate |
| Port other than 8080 | Use user-specified port in both STEP.txt and Dockerfile EXPOSE |
| PRODUCTION environment | Use `PRODUCTION` in ZIP name; remind user to double-check WSDL URL before handover |
| User skips intake form fields | Ask for missing required fields before proceeding — do not use guessed values |
| WAR context path differs from container name | Use `[WAR_CONTEXT]` only for the URL path; `[CONTAINER]` only for `docker run --name` |

---

## Level History

- **Lv.1** — Base: Intake form → verify folders → create Dockerfile → create STEP.txt → ZIP → summary. Standard MTSA tomcat:9-jdk17-corretto Docker template. (Origin: 2026-06-09, based on AMANAH KREDIT PILOT deployment package pattern)
