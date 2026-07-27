---
name: dotnet-native-crash-diagnosis
description: >
  Use this skill when a .NET Framework/WPF/WinForms app crashes with no managed exception logged —
  the process just dies, or crashes only on specific customer machines. Trigger when user mentions
  a native crash, WER dump analysis, WinDbg, "STATUS_INVALID_EXCEPTION_HANDLER" or similar 0xc00...
  codes, PKCS#11/smart-card driver crashes, AnyCPU vs x86 vs x64 confusion, Prefer32Bit, WOW64,
  or a Visual Studio Installer (.vdproj) install-path/architecture mismatch.
---

# .NET Native Crash Diagnosis — Dejul's Practice
*Root-causing crashes the CLR never sees, and locking a project's bitness to stop them for good*

## Why this exists

Managed exception handlers (`try/catch`, even `[HandleProcessCorruptedStateExceptions]` and
`AppDomain.UnhandledException`) do **not** catch native SEH-chain corruption faults
(`STATUS_INVALID_EXCEPTION_HANDLER`, `0xc00001a5`, and similar). These originate below the CLR,
typically inside a third-party native DLL (PKCS#11 drivers, smart-card minidrivers, etc.) — the
process just vanishes with zero application-level log output. Standard debugging is useless here;
you need the actual crash dump and a native debugger.

## Protocol

### Step 1: Capture a crash dump
- Ensure Windows Error Reporting (WER) is configured to keep local dumps, or reproduce with a
  debugger attached.
- Get the `.dmp` file from the machine that actually crashes — a "works on my machine" comparison
  is often the fastest path to root cause (see Step 4).

### Step 2: Analyze with WinDbg
- Install if missing: `winget install Microsoft.WinDbg` (installs WinDbg Preview; bundles
  `cdb.exe` at `...\Microsoft.WinDbg_<version>\amd64\cdb.exe`).
- Run: `cdb.exe -z <path-to-dump> -c "!analyze -v; ~*k; q"` (or open interactively and run
  `!analyze -v` then `~*k` for all-thread stacks).
- Key fields to pull from `!analyze -v` output:
  - `Failure.ProblemClass.Primary` / `ExceptionCode` — e.g. `BITNESS_MISMATCH_X86_INVALID_EXCEPTION_HANDLER`
  - `CLR.BitnessMismatch` / `OSPLATFORM_TYPE` — confirms whether the process ran x86 (WOW64) or x64
  - The crashing thread's full stack — look for `<Unloaded_xxx.dll>` frames just before the fault;
    WinDbg may flag frames past a corruption point as "not in any known module" — expected, the
    module identity itself is still trustworthy even if the unwind is unreliable.
- Note: unloaded-module entries in a dump only retain base address/size, not PE version info — if
  you need exact DLL versions, pull them live from the machine (`Get-Item file | select VersionInfo`),
  not from the dump.
- **Security note**: full dumps can contain plaintext secrets (PINs, passwords) in memory — keep
  local, delete after analysis, don't attach to tickets/emails.

### Step 3: Verify actual bitness — never assume from folder name
`SysWOW64` = 32-bit compat, `System32` = native 64-bit (yes, this is backwards-sounding — verify,
don't guess). To confirm a DLL or EXE's real architecture, read the PE header `Machine` field
directly (`0x8664` = x64, `0x014c` = x86), or use `CorFlags.exe` (Windows SDK) for managed
assemblies — look for `PE32` (32-bit or AnyCPU) vs `PE32+` (64-bit) and the `32BITPREF`/`32BITREQ`
flags.

`PlatformTarget=AnyCPU` + `Prefer32Bit=false` is **not** the same as `PlatformTarget=x64` — AnyCPU
adapts to the host OS (would still run 32-bit on an actual 32-bit Windows). If the whole project is
being committed to 64-bit only, set `PlatformTarget=x64` explicitly in every `PropertyGroup` that's
actually built — `Prefer32Bit` becomes a meaningless no-op once `PlatformTarget` is explicit and
should be removed, not left in place.

### Step 4: Compare against a reference tool if one exists
If another tool uses the same native library/driver and doesn't crash, diff its project settings
(bitness, library version) against yours — this is often stronger evidence than the dump alone.
Example: comparing against Pkcs11Admin (reference PKCS#11 GUI tool, same library author) showing
it runs native x64 and never crashes corroborated that WOW64 itself — not a specific driver
version — was the real hazard.

### Step 5: Installer (.vdproj / MSI) architecture is independent of the app's bitness
A Visual Studio Installer project's `TargetPlatform` property (`"3:0"`=x86, `"3:1"`=x64) controls
the **MSI package's** architecture — which resolves `[ProgramFilesFolder]` (`Program Files` vs
`(x86)`) and which registry view it uses — separately from what the bundled `.exe` targets. An x86
MSI can still install fine on a 64-bit OS even if the exe inside now requires 64-bit, producing a
confusing "installed OK, crashes on launch" experience instead of a clear install-time failure.

To verify what a *built* MSI actually is (don't just trust the `.vdproj` source text — Windows
Installer's major-upgrade path-preservation can make a fresh x64 MSI still install into an old x86
`TARGETDIR` if one exists on the machine):
```powershell
$installer = New-Object -ComObject WindowsInstaller.Installer
$db = $installer.GetType().InvokeMember("OpenDatabase", "InvokeMethod", $null, $installer, @("path\to\file.msi", 0))
$si = $db.GetType().InvokeMember("SummaryInformation", "GetProperty", $null, $db, @(0))
$si.GetType().InvokeMember("Property", "GetProperty", $null, $si, @(7))  # Template: "x64;1033" or "Intel;1033"
```

If dropping x86 support project-wide: update the `.csproj`(s) (`PlatformTarget=x64`), the `.vdproj`
(`TargetPlatform` → `"3:1"`), and any install/uninstall automation scripts — during the migration
period, scripts must check **both** old (`Program Files (x86)`, `WOW6432Node` registry) and new
(`Program Files`, native registry) locations, since machines may still carry an old x86-packaged
install that needs to be found and removed.

### Step 6: Process isolation — the only real mitigation for native SEH faults
If a specific native call can't be made safe (can't get a reliable driver fix, or bitness can't be
changed yet), the only architecture that actually contains a native SEH fault is running that call
in a **separate child process** (`System.Diagnostics.Process.Start`). **AppDomains do NOT protect
against this** — a native fault takes down the whole process regardless of AppDomain boundaries.
Design such isolation to be a thin, behavior-preserving wrapper: same inputs/outputs, gated behind
a config flag defaulting to the pre-existing in-process behavior, so no existing caller's behavior
changes unless the flag is explicitly flipped on.

### Step 7: `pkcs11-logger` for call-level PKCS#11 tracing
When bitness/dump analysis narrows the problem to "somewhere inside PKCS#11 driver calls" but not
which call, use `pkcs11-logger` (github.com/Pkcs11Interop/pkcs11-logger) — a proxy DLL that logs
every PKCS#11 call with parameters/return values.
- Match the logger DLL's bitness to the process (verify via PE header, same as Step 3).
- Set env vars `PKCS11_LOGGER_LIBRARY_PATH` (the real driver DLL it wraps) and
  `PKCS11_LOGGER_LOG_FILE_PATH`, then point the app's config at the **logger** DLL instead of the
  real driver.
- **Gotchas that cause "no log file generated at all"**: env vars set via `setx` don't affect the
  currently-running session/already-cached environment blocks — the machine needs a reboot or at
  minimum a fresh logon before launching the app; and the edit must be made to the **deployed**
  config file the running exe actually reads (e.g. `MyApp.exe.config` next to the installed exe),
  not the source-repo `App.config`, with the app fully relaunched afterward (`ConfigurationManager`
  caches per-AppDomain).

## Level History
- **Lv.1** — Base: WinDbg dump capture/analysis workflow, PE-header bitness verification (never
  trust folder names), AnyCPU-vs-explicit-x64 distinction, MSI `SummaryInformation.Template`
  verification via COM, x86→x64 migration handling for install scripts, process isolation as the
  only real native-fault mitigation, `pkcs11-logger` setup + its common silent-failure causes.
  Source: MyTrustID Desktop sessions 18-20 (2026-07-19 to 2026-07-27) — a PKCS#11 smart-card token
  crash traced to `STATUS_INVALID_EXCEPTION_HANDLER` under WOW64, resolved by committing the whole
  project to x64-only plus Phase 1 process isolation for token detection.
