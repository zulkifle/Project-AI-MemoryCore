# Windows Server Housekeeping - Log Cleanup Batch Script
*Automated .bat script to purge log files older than 10 days from Trustgate server log directories*

## Project Overview
- **Type**: Tool (Windows Batch Script)
- **Client**: Trustgate Sdn Bhd (Internal)
- **Period**: 2026-04-29 - Active
- **Tech Stack**: Windows Batch Script (.bat) + forfiles + Windows Task Scheduler
- **Completion**: 0%
- **Duration**: 0 min
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-04-29 - Project scoped, paths confirmed, approach planned
- **Next Steps**: Write housekeeping.bat using forfiles — user to provide save path on server, then deliver script
- **Known Issues**: Save path for the .bat file not yet confirmed by user

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

### 2026-04-29 - Project Created
- **Changes**: Initial project setup, log paths defined
- **Time Spent**: ~0 min

## Historical Summary
[No history yet — this section is populated when session count exceeds 5]

## Technical Notes
- **Repository**: TBD (local script, may be placed on server directly)
- **Key Dependencies**: `forfiles` (built-in Windows command), Windows Task Scheduler
- **Deployment**: Copy .bat to server, schedule via Task Scheduler (daily, off-peak hours)

---
**Last Updated**: 2026-04-29 | **Position**: #1/10 Active
