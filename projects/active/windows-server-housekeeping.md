# Windows Server Housekeeping - Log Cleanup Batch Script
*Automated .bat script to purge log files older than 10 days from Trustgate server log directories*

## Project Overview
- **Type**: Tool (Windows Batch Script)
- **Client**: Trustgate Sdn Bhd (Internal)
- **Period**: 2026-04-29 - Active
- **Tech Stack**: Windows Batch Script (.bat) + forfiles + Windows Task Scheduler
- **Completion**: 100%
- **Duration**: 30 min
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-05-25 - Script written and saved ✅
- **Next Steps**: Copy housekeeping.bat to Trustgate server → schedule via Task Scheduler (daily 2:00 AM)
- **Known Issues**: None — script ready to deploy

## Log Paths (4 targets)
- `C:\wildfly-18.0.1.Final\standalone\log`
- `C:\Trustgate\MyTrustSignServer\logs`
- `C:\Trustgate\MyTrustSignServer\rsc\logs`
- `C:\Trustgate\MyTrustSignServer\DigitalSeal\log`

## Rules
- Retain last **10 days** of logs
- Delete files older than 10 days
- Table is **read-only** — no application files touched, log folders only

## Session History (Last 5)

### 2026-05-25 - Script Written & Saved
- **Changes**: housekeeping.bat written — 4 log paths, 10-day retention, forfiles recursive delete, audit log to C:\Trustgate\housekeeping-report.log. Saved to C:\PROJECTS\ECOURT\SIGN SERVER ARCH DIAGRAM\Release v2 ecourt\
- **Time Spent**: ~30 min

### 2026-04-29 - Project Created
- **Changes**: Initial project setup, log paths defined
- **Time Spent**: ~0 min

## Historical Summary
[No history yet — this section is populated when session count exceeds 5]

## Technical Notes
- **Repository**: `C:\PROJECTS\ECOURT\SIGN SERVER ARCH DIAGRAM\Release v2 ecourt\housekeeping.bat`
- **Key Dependencies**: `forfiles` (built-in Windows command), Windows Task Scheduler
- **Deployment**: Copy .bat to server → schedule via Task Scheduler (daily 2:00 AM)
- **Audit log on server**: `C:\Trustgate\housekeeping-report.log`

---
**Last Updated**: 2026-04-29 | **Position**: #1/10 Active
