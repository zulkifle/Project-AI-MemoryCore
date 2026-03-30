# 🔄 Sync Dev Source & Deployment Config

**Category**: Deployment
**Type**: Workflow Pattern
**Tested on**: Java/NetBeans + Tomcat projects (MTSA JKR)

---

## The Problem

In projects with separate **development source** and **deployment folders**, config files exist in two places:

| Location | Used when |
|---|---|
| `Development/src/` | NetBeans Clean & Build copies this into `build/web/WEB-INF/classes/` |
| `Deployment/WEB-INF/classes/` | The actual running server reads this |

**Risk**: If you only update the deployment folder, a Clean & Build will **overwrite** it with the stale source version — silently reverting your changes.

---

## The Rule

> **Always update BOTH locations when changing config files.**

Common files affected:
- `log4j.properties` — log file paths change per environment
- `mtsagent.properties` — `ws.appdir` path changes per environment

---

## The Pattern — switch-env script

Use a `switch-env` script (PowerShell or batch) that updates both locations at once.

### PowerShell structure

```powershell
$BASE    = "C:\...\Deployment"
$DEV_SRC = "C:\...\Development\myproject\src\java"

function Set-Env([string]$logMain, [string]$workdir) {

    # 1. Update deployed WEB-INF/classes/
    ReplaceProp "$BASE\WEB-INF\classes\log4j.properties" "log4j.appender.X.File" $logMain
    ReplaceProp "$BASE\WEB-INF\classes\mtsagent.properties" "ws.appdir" $workdir

    # 2. Sync NetBeans dev source — prevents Clean & Build from reverting
    ReplaceProp "$DEV_SRC\log4j.properties" "log4j.appender.X.File" $logMain
    ReplaceProp "$DEV_SRC\mtsagent.properties" "ws.appdir" $workdir
}
```

### ReplaceProp helper (handles backslashes safely)

```powershell
function ReplaceProp([string]$file, [string]$key, [string]$value) {
    if (-not (Test-Path $file)) { Write-Warning "NOT FOUND: $file"; return }
    $content = Get-Content $file -Raw
    $pattern = "(?m)^$([regex]::Escape($key))=.*$"
    $content = [regex]::Replace($content, $pattern, { param($m) "$key=$value" })
    [System.IO.File]::WriteAllText($file, $content)
}
```

### Batch launcher (double-click friendly)

```batch
@echo off
echo  [1] localhost
echo  [2] sandbox / server
set /p CHOICE="Select: "
if "%CHOICE%"=="1" set ENV=localhost
if "%CHOICE%"=="2" set ENV=sandbox
powershell -ExecutionPolicy Bypass -File "%~dp0switch-env.ps1" %ENV%
pause
```

---

## Notes

- Files that are **runtime-only** (not bundled in WAR) only need the deployment copy updated — e.g. `gpki.properties` read via `ws.appdir` path.
- Files **bundled in WAR** (classpath resources) need both copies updated.
- Always run `switch-env` **before** Clean & Build in the IDE.
