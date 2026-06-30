# Petronas Legacy C++ Dockerize
*Legacy C++ daemon (PetronasService) — downloads card data via SFTP, signs with SafeNet HSM (PKCS#11), uploads signed files back to Petronas*

## Project Overview
- **Type**: Legacy C++ Dockerization
- **Client**: Petronas (Handover Project)
- **Period**: 2026-06-03 - Active
- **Tech Stack**: Backend: C++ (g++) | DB: MySQL (PetronasChipsCard) | Container: Docker (debian:bookworm) | Signing: SafeNet Luna HSM (PKCS#11/Crystoki) | OpenSSL CLI
- **Completion**: 80%
- **Duration**: ~135 min
- **Due Date**: 2026-06-05 ⚠️ OVERDUE — still in test phase
- **Source Path**: `C:\PROJECTS\HANDOVER PROJECT\Petronas\source code\amg backup as per 29-06-2026\`

## Current Status
- **Last Session**: 2026-07-01 - DB connection fixed (MySQL Router via SSH tunnel)
- **Next Steps**:
  1. Start SSH tunnel: `ssh -i ~/.ssh/zul_rsakey -L 3306:127.0.0.1:3306 zul@10.5.1.42`
  2. Run `docker compose up` — verify logs show DB connected + "Start Loop"
  3. Test TCP port: `Test-NetConnection localhost -Port 6803`
  4. When ready for production: swap commented volume blocks in docker-compose.yaml + change DB Host to MySQL Router IP
  5. On production Linux server: mount Crystoki lib + SSH keys
- **Known Issues**:
  - HSM (SafeNet Crystoki) not available in Docker — must set `HSM=0` for local test
  - SFTP SSH keys not mounted (commented out in docker-compose)
  - `openssl rsautl` deprecated in OpenSSL 3.x (container) — still works, prod uses `openssl1` alias

## Session History (Last 5)

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
- **DB connection (PROD)**: Change `Host` in `Petronas.ini` to MySQL Router IP (e.g. `127.0.0.1` if local on server)

---
**Last Updated**: 2026-07-01 | **Position**: #1/10 Active
