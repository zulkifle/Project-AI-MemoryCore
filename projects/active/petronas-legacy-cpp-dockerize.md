# Petronas Legacy C++ Dockerize
*Legacy C++ daemon (PetronasService) — downloads card data via SFTP, signs with SafeNet HSM (PKCS#11), uploads signed files back to Petronas*

## Project Overview
- **Type**: Legacy C++ Dockerization
- **Client**: Petronas (Handover Project)
- **Period**: 2026-06-03 - Active
- **Tech Stack**: Backend: C++ (g++) | DB: MySQL (PetronasChipsCard) | Container: Docker (debian:bookworm) | Signing: SafeNet Luna HSM (PKCS#11/Crystoki) | OpenSSL CLI
- **Completion**: 87%
- **Duration**: ~165 min
- **Due Date**: 2026-06-05 ⚠️ OVERDUE — still in test phase
- **Source Path**: `C:\PROJECTS\HANDOVER PROJECT\Petronas\source code\amg backup as per 29-06-2026\`

## Current Status
- **Last Session**: 2026-07-22 - Analyzed production log for two distinct issues (see below) — read-only investigation, no code changes
- **Previous Session**: 2026-07-01 - Fixed localhost socket error → host.docker.internal
- **Next Steps**:
  1. `docker compose down && docker compose up` — no rebuild needed (ini is volume-mounted)
  2. Verify logs show DB connected + "Start Loop"
  3. Test TCP port: `Test-NetConnection localhost -Port 6803`
  4. PROD deploy: uncomment `mysqlrouter.mysqlrouter:6446` block in ini + swap volume blocks in docker-compose
  5. **NEW**: Investigate DB directly — `SELECT * FROM pm_staticmaster WHERE pmsm_id=3069;` + `SELECT * FROM pm_staticdetail WHERE pmsd_smid=3069;` — confirm orphaned record, decide fix (repopulate detail rows vs. reset `pmsm_manual` flag vs. archive/delete)
- **Known Issues**:
  - HSM (SafeNet Crystoki) not available in Docker — must set `HSM=0` for local test
  - SFTP SSH keys not mounted (commented out in docker-compose)
  - `openssl rsautl` deprecated in OpenSSL 3.x (container) — still works, prod uses `openssl1` alias
  - **Port 6803 `EADDRINUSE` crash-loop**: app logs show repeated `APPLICATION START` → `Error while binding the socket: 98 [Address already in use]` → `APPLICATION ENDED` every ~5 min (17:10, 17:15, 17:20 in the 2026-07-21 log). Something else already holds port 6803 — likely a previous instance/container that wasn't cleanly stopped before the next one started. Not yet root-caused to a specific process; check for a lingering container/process on the host before next restart.
  - **`pm_staticmaster` record 3069 permanently stuck** (found 2026-07-22, see below) — orphaned master row with zero matching `pm_staticdetail` rows; the code has no retry/error path for this case, so it silently blocks forever at the "manual" verification stage.

## Session History (Last 5)

### 2026-07-22 - Log Analysis: -13 Errors (Benign) + Stuck Record 3069 (Real Bug)
- **Changes**: Read-only investigation of a production log Dejul pasted, no code changes.
  - **`DB_GetStaticMaster(...,0,5,0)` / `(...,0,6,0)` returning `-13`**: traced to `DB.cpp:1095-1101` — `-13` means the `SELECT COUNT(*) FROM pm_staticmaster WHERE pmsm_upload='N'` (or `pmsm_done='N'`) query returned zero rows, i.e. "nothing pending at this stage right now." Logged at severity 5 with "xxx Failed," which is misleading — it's the normal empty-queue case, not an error. No action needed (same misleading-log-severity pattern previously flagged in MyTrustID Desktop's "OnStartUp Exception" line).
  - **`VerifySSAD2(3069)` returning `-1`, root cause `DB_GetStaticDetailList(3069)` returning `-9`**: traced to `DB.cpp:1279` (`iFlag` stays at its initial `-9` because the `SELECT pmsd_id, pmsd_name FROM pm_staticdetail WHERE pmsd_smid=3069` query found zero matching rows) — master record 3069 in `pm_staticmaster` has no child rows in `pm_staticdetail` at all, so there's nothing to verify.
  - **The actual bug**: in `SSAD.cpp:98-115`, when `VerifySSAD2()` fails, the code only logs the failure — it never calls `DB_SetFlagMaster(iReference, 4, 2)` (success) or any failure-state transition. Since `DB_SetFlagMaster(3069, 4, 1)` was called just before (marking `pmsm_manual='P'`, in-progress) and `DB_GetStaticMaster(...,0,4,0)` only re-selects rows where `pmsm_manual='N'`, record 3069 is now **permanently stuck** at "in progress" for the manual-verification stage — no retry, no error flag, silently ignored on every future loop.
  - Gave Dejul the DB queries to confirm the orphan and next-step options (repopulate detail rows / reset the flag to retry / archive the dead record) — decision pending his call on what actually happened to that batch's data.
- **Time Spent**: ~15 min

### 2026-07-01 - Fixed localhost Unix Socket Error
- **Changes**:
  - Error: `Connect fails 2002 [Can't connect to local server through socket '/run/mysqld/mysqld.sock']`
  - Root cause: MySQL C client on Linux treats `localhost` as Unix socket (not TCP), ignoring Port=3306
  - Fix: Changed `Host="localhost"` → `Host="host.docker.internal"` in `Petronas.ini`
  - No rebuild needed — ini is volume-mounted, `docker compose down && docker compose up` is enough
  - Rule: inside Docker container on Linux, always use `host.docker.internal` (not `localhost`) to reach Windows host MySQL via TCP
- **Time Spent**: ~5 min

### 2026-07-01 - DB Switched to localhost root for Local Test
- **Changes**:
  - `Petronas.ini` [Database] restructured into 3 clearly labelled blocks (only one active at a time):
    - **LOCAL TEST** (active): `Host="localhost"`, `Port=3306`, `User="root"`, `Password="7ru57ga73"`
    - **Docker TEST** (commented): `host.docker.internal:3306`, `chips`/`dbaadmin` via SSH tunnel
    - **PRODUCTION K8s** (commented): `mysqlrouter.mysqlrouter:6446`, `chips`/`dbaadmin`
  - Note: `localhost` only works running binary directly — inside Docker use `host.docker.internal`
- **Time Spent**: ~5 min

### 2026-07-01 - DB Port Configurable + K8s MySQL Router Support
- **Changes**:
  - Discovered `DB_Open()` had port hardcoded as `3306` — `iPort` variable was declared in `DB_ReadConfig()` but never used (original dev left a placeholder)
  - Modified both `Sources_29072024/DB.cpp` and `Sources/DB.cpp`:
    - Added `static unsigned int ui_DBPort = 3306` at module level
    - `DB_ReadConfig()` now reads `Port` from `[Database]` section in ini (defaults to 3306 if missing)
    - `DB_Open()` now passes `ui_DBPort` to `MySql.Register()` instead of hardcoded 3306
  - Updated `Petronas.ini` `[Database]` section: added `Port=3306` (TEST), comments for `Host="mysqlrouter.mysqlrouter"` + `Port=6446` (PRODUCTION K8s)
  - Noted K8s kubectl access: `ssh -L 6443:10.5.1.42:6443 zul@10.5.1.42` — kubeconfig must have `server: https://127.0.0.1:6443`
  - **Rebuild required** after DB.cpp changes: `docker compose build --no-cache`
- **Time Spent**: ~20 min

### 2026-07-01 - DB Connection via MySQL Router (SSH Tunnel)
- **Changes**:
  - Updated `Softwares/Petronas/Petronas.ini` — DB `Host` changed from `10.3.9.51` (old direct IP) to `host.docker.internal` (resolves to Windows host where SSH tunnel is active on port 3306)
  - Added comments in `Petronas.ini` explaining TEST (requires SSH tunnel) vs PRODUCTION (use MySQL Router IP directly) configuration
  - Added `extra_hosts: - "host.docker.internal:host-gateway"` to `docker-compose.yaml` — required for Linux Docker Engine compatibility (Docker Desktop resolves it automatically on Windows)
  - Flow: Container → `host.docker.internal:3306` → Windows host SSH tunnel → `127.0.0.1:3306` on 10.5.1.42 → MySQL Router → DB
- **Time Spent**: ~15 min

### 2026-06-30 - Dockerized + Mermaid Diagrams
- **Changes**:
  - Explored full source: `PetronasService` (C++ daemon), `HSM.cpp`, `FTP.cpp`, `SSAD.cpp`, `DB.cpp`, `GenerateSSAD.cpp`, CronShell scripts, `Petronas.ini`
  - Generated `docs/petronas-diagrams.md` — 6 Mermaid diagrams (system overview, main loop, FTP flow, SSAD generation, infra, monitoring & alerting)
  - Created `Dockerfile` — multi-stage build: g++ compile from source (debian:bookworm builder) → minimal runtime with libmariadb3 + openssh-client + openssl
  - Created `docker-compose.yaml` — TEST mode (Windows relative paths `./testdata/` + `./Softwares/Petronas/Petronas.ini`) vs PRODUCTION mode (Linux `/opt/petronas/...`) — swap comments to switch
  - Created `entrypoint.sh` — validates Petronas.ini present before starting service
  - Created `STEP.txt` — full deployment guide (host dirs, SSH keys, HSM, build, run, verify)
  - `testdata/` folder structure created for Windows local testing
  - Fixed: `openssl` missing from Dockerfile runtime — added
  - Saved TEST vs PROD path switching to Claude auto-memory
- **Time Spent**: ~120 min

### 2026-06-03 - Project Created
- **Changes**: Initial project setup. Source not yet retrieved.
- **Time Spent**: ~0 min

## Historical Summary
[No history yet — this section is populated when session count exceeds 5]

## Technical Notes
- **Repository**: `C:\PROJECTS\HANDOVER PROJECT\Petronas\source code\amg backup as per 29-06-2026\`
- **Key Dependencies**: g++ (build), libmariadb3 (runtime), openssl CLI (runtime shell calls), openssh-client (SFTP), SafeNet Crystoki PKCS#11 lib (HSM)
- **Architecture**: C++ daemon listening on TCP :6803. Main loop every 60 min: FTPIn → FTPExtract → SSADProcess (HSM) → FTPOut. SFTP alias `petronas` configured via ~/.ssh/config.
- **Crystoki lib paths**: `/usr/lib/libCryptoki2_64.so` (Luna 5.x) or `/usr/lib/libCryptoki2.so` (Luna 6.x+)
- **INI file**: read from working dir `./Petronas.ini` — `HSM=0` under `[System]` disables HSM for testing
- **docker-compose TEST vs PROD**: See auto-memory `project_petronas_docker.md` for exact lines to swap
- **TCP health check**: send `TESTHSM` → get `ALIVE` or `DEATH`; send anything else → get current datetime
- **DB connection (TEST)**: `host.docker.internal:3306` via SSH tunnel `ssh -L 3306:127.0.0.1:3306 zul@10.5.1.42`
- **DB connection (PROD K8s)**: `Host="mysqlrouter.mysqlrouter"` + `Port=6446` in `Petronas.ini` — matches JDBC: `jdbc:mysql://mysqlrouter.mysqlrouter:6446/PetronasChipsCard`
- **DB port**: now read from `[Database]` → `Port=` in ini (was hardcoded 3306 in DB.cpp — both copies fixed)
- **K8s access**: `ssh -L 6443:10.5.1.42:6443 zul@10.5.1.42` — kubeconfig `server:` must be `https://127.0.0.1:6443`

---
**Last Updated**: 2026-07-01 | **Position**: #1/10 Active
