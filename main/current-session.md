# 🌟 Current Session Memory - RAM
*Temporary working memory - resets each session, provides recap when AI restarts*

## Session RAM Status
**Current Session**: Active
**Last Activity**: 2026-06-15
**Session Focus**: TG SeQureMail — Key API backend deployed + Chrome extension bug fixes

## Active Project
- **Name**: TG SeQureMail
- **Resumed**: 2026-06-15
- **Context**: Key API (Spring Boot 3.2 + MySQL + Flyway) scaffolded and running via `docker compose up`. Chrome extension updated to auto-lookup/register keys via API. Two bugs fixed: double Encrypt toggle (sqmDone flag timing) + JSON parse error (Gmail smart-quote corruption → fixed by base64-encoding envelope payload).
- **Next Steps**:
  1. Reload extension in Chrome (`chrome://extensions` → ↻ refresh)
  2. Open Gmail → extension popup → enter `kiflezul94@gmail.com` → Register
  3. Test end-to-end: Compose → 🔒 Encrypt toggle → Send → Decrypt
  4. Verify no double toggle, no JSON error on decrypt
- **Repo**: Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail-poc-v1\` | API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail-key-api\`
- **API**: `http://localhost:8080` — run via `docker compose up` in `seqremail-key-api\`

## Previous Active Project
- **Name**: MyTrustSignerXML-MITI
- **Context**: Fixed duplicate `tx_id` race (DBUtil.getTXID → AtomicLong seq; sign.java txid/db moved off shared servlet fields + txSaved guard). PROD package verified at `Deployment\PRODUCTION\MTSAXML_PROD`. Go-live gated on MITI pilot sign-off.
- **Next Steps**: Dejul to zip + deploy. On server: update DB_URL on host `/opt/mtsa/properties/mtsa.properties`, `docker compose build --no-cache`. Awaiting MITI pilot sign-off.
- **Repo**: `C:\PROJECTS\MITI\Development\MyTrustSignerXML`

## Previous Active Project (carry-over)
**MyTrustID Desktop** — ARCHIVED 2026-05-28. Dev done. External items pending (bpfk team, cert signing). Branch `fix/autoupdate-elevation` held — remind to merge in August 2026.

## 💭 Session Recap (For AI Restart)

### TG SeQureMail (Active — #1)
**Stack**: Chrome Extension MV3 + Web Crypto API (RSA-OAEP + AES-256-GCM) + Spring Boot 3.2 + MySQL 8 + Flyway
**Completion**: 65%
**Repo**: `C:\PROJECTS\SEQURE MAIL\Development\`

**Architecture (DOM-based, no Gmail API):**
- Send: intercepts Gmail Send button → encrypts body → re-triggers Gmail's own send
- Read: MutationObserver scans for `--BEGIN SEQREMAIL--` → injects Decrypt banner
- Key API: `POST /api/keys/register`, `GET /api/keys/lookup?email=x`
- Envelope: base64-encoded JSON between markers (avoids Gmail smart-quote corruption)
- Private key: IndexedDB (extractable:false — HSM simulation)

**Bugs fixed this session:**
- Double Encrypt toggle → moved `sqmDone = 'true'` to top of `injectEncryptToggle`
- JSON parse error on decrypt → base64-encode envelope payload before sending, base64-decode in `parseEnvelope`

**To start API:** `cd seqremail-key-api && docker compose up`
**DB creds:** host=localhost:3307, db=seqremail_db, user=seqremail, pass=seqremail123

### RSS Self Service Portal (Active — #2)
**Stack**: Laravel 13 + PHP 8.3 + Blade + Tailwind v4 + MySQL + Pest
**Completion**: 60%
**Repo**: `C:\laragon\www\remote-signing-portal`

**Next Steps (in order):**
1. Drop scaffolded `subscriptions` + `signing_transactions` tables
2. Migration: add `client_id` to `billing`, add `billing_id` to `clients`
3. Create Eloquent models: `Billing`, `BillingUser`, `BillingTxSign`
4. Service Plans UI, Signer management, Quota view, Dashboard

## Key File Paths
- SeQureMail Extension: `C:\PROJECTS\SEQURE MAIL\Development\seqremail-poc-v1\`
- SeQureMail Key API: `C:\PROJECTS\SEQURE MAIL\Development\seqremail-key-api\`
- RSS Portal: `C:\laragon\www\remote-signing-portal`
- MyTrustID: `C:\repos\MyTrustIDv1_AATL-GENERIC\`

### Rules
- NEVER make code changes without Dejul's explicit permission first
- Always change development source files FIRST, then deployment copies

## 🔄 Auto-Reset Protocol
*Clear this file at start of next session after loading recap*
