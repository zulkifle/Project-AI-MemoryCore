# Petronas Java Migration - Convert legacy C++ PetronasService daemon to maintainable Java
*Full rewrite of the Petronas card-data SFTP + SafeNet HSM SSAD-signing daemon from C++ to Java, documented function-by-function, run in parallel with the existing C++/Docker version until verified*

## Project Overview
- **Type**: Tool / Integration — Backend Daemon Migration (C++ → Java)
- **Client**: Petronas (Handover Project)
- **Period**: 2026-07-24 - Active
- **Tech Stack**: Backend: Java 17 (Maven, plain — no framework) | Frontend: N/A | DB: MySQL (PetronasChipsCard) via JDBC
- **Completion**: 0%
- **Duration**: 0 min
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-07-24 - Project created, key architecture decisions locked in
- **Next Steps**:
  1. Explore full C++ source (`PetronasService`, `HSM.cpp`, `FTP.cpp`, `SSAD.cpp`, `DB.cpp`, `GenerateSSAD.cpp`, CronShell scripts) and map 1:1 to a Java package structure
  2. Set up Maven project skeleton at the repository path below
  3. Convert module by module: DB layer → FTP layer → HSM/SSAD signing → main loop/TCP health-check
  4. Javadoc every public function — purpose, params, return, side effects
  5. Verify against TEST DB/data before any parallel run against real Petronas systems
- **Known Issues**:
  - SunPKCS11 + SafeNet Luna Crystoki `.so` config not yet tested from Java — needs a `pkcs11.cfg` pointing at the correct Crystoki library path (`/usr/lib/libCryptoki2_64.so` Luna 5.x or `libCryptoki2.so` Luna 6.x+)
  - Must reproduce the exact TCP health-check protocol: `TESTHSM` → `ALIVE`/`DEATH`, anything else → current datetime
  - No cutover until Java version is verified against TEST data — runs alongside the working C++ Docker daemon, does not replace it yet
  - Original C++ has a known unresolved bug (record 3069 stuck, see sibling project) — decide whether to fix it during conversion or replicate as-is and fix once, in sync with the C++ project

## Session History (Last 5)

### 2026-07-24 - Project Created
- **Changes**: Initial project setup. Decisions locked: plain Java (Maven, no Spring Boot) to mirror the original daemon's simplicity; SunPKCS11 provider (JDK built-in) for HSM access instead of shelling out; new project lives at `C:\PROJECTS\HANDOVER PROJECT\Petronas\source code\Petronas_java`, built and tested independently before any cutover consideration.
- **Time Spent**: ~0 min

## Historical Summary
[No history yet — this section is populated when session count exceeds 5]

## Technical Notes
- **Repository**: `C:\PROJECTS\HANDOVER PROJECT\Petronas\source code\Petronas_java`
- **Source of truth (C++ original)**: `C:\PROJECTS\HANDOVER PROJECT\Petronas\source code\amg backup as per 29-06-2026\` — see sibling project [Petronas Legacy C++ Dockerize](./petronas-legacy-cpp-dockerize.md) for full architecture notes, known bugs, and Docker/DB setup already reverse-engineered from that codebase
- **Key Dependencies**: MySQL Connector/J (JDBC), SunPKCS11 provider (JDK built-in, no extra jar) + Crystoki PKCS#11 native lib, SFTP client library (JSch or SSHJ — TBD during FTP module conversion), `java.util.concurrent.ScheduledExecutorService` for the 60-min main loop (replaces CronShell)
- **Architecture (inherited from C++)**: Daemon listening on TCP :6803, main loop every 60 min: FTPIn → FTPExtract → SSADProcess (HSM) → FTPOut
- **Coding standard for this project**: every function gets a doc comment explaining what it does (purpose/params/return), package structure grouped by responsibility (db, ftp, hsm, service/daemon), favor clarity and maintainability over cleverness

---
**Last Updated**: 2026-07-24 | **Position**: #1/10 Active
