# TradeVault - Self-Hosted Personal Trading OS
*"One Platform. Every Trade. Every Market." — Modular monolith trading journal/analytics platform for Dejul's personal use.*

## Project Overview
- **Type**: Web App
- **Client**: Internal (personal use)
- **Period**: 2026-07-22 - Active
- **Tech Stack**: Backend: Spring Boot 3 + Java 21 + Spring Security (JWT) | Frontend: React + TypeScript + Vite + TailwindCSS + shadcn/ui + React Router + React Hook Form + Recharts | Database: PostgreSQL (Spring Data JPA)
- **Completion**: 85%
- **Duration**: ~12 hours (estimated — no Auto-Commit time tracking; TradeVault has no git repo of its own yet)
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-08-26 - Designed + built Feature B (moomoo order-submission: EMAS Calculator + auto TP/SL + self-built OCO) end-to-end, extending Feature A's `moomoo-bridge` in place. Backend compiles, frontend builds, bridge syntax-checked. Neither Feature A nor B has been run live yet — both need Dejul's manual deployment steps.
- **Next Steps**:
  1. **Deploy Feature A + B together** (they share one bridge process): `docker compose up -d --build`, create the "moomoo MY (REAL)" Account + Calculation Profile in the UI, fill in `accounts.json`'s account UUID + `moomoo-bridge/.env`, `pip install -r requirements.txt` (now includes fastapi/uvicorn), run `python bridge.py`. Verify Feature A first (place a trade via Claude Code as usual, confirm it journals), then Feature B (submit-only dry run first — see its section below — before letting a real entry fill).
  2. Continue real usage — log actual trades, verify charts/reports/analytics against real data
  3. Consider `git init` for the TradeVault app repo itself (separate from Jessy's tracking repo)
  4. Add `frontend/eslint.config.js` (the `lint` npm script currently has no config to run against)
  5. V2 scope, when ready: add calculators for Bursa/Crypto/Futures/etc. per the pluggable `TradeCalculator` pattern — no Trade Engine changes needed; Feature B's SELL-side flow, Equity-Based sizing, reverse calculators
  6. The Firefox extension idea (auto-fill Entry/SL/TP from a chart into the Order Ticket) remains a fully separate, unstarted project
- **Known Issues**: None currently open. `smoketest` account + one `XM Demo` (FOREX) account exist in the dev DB from verification testing — harmless, safe to delete or ignore.

## Feature A: moomoo Journal Auto-Sync Bridge — Built, Not Yet Deployed
*Spec: `docs/specs/2026-08-25-moomoo-journal-sync-design.md`. Code written and compiling/syntax-checked 2026-08-25; never run against a live TradeVault instance yet.*

**What was built:**
- Backend (compiles clean via `mvn compile`, frontend via `npm run build`): `StockCalculator` (`STOCK_STANDARD`, reuses the existing `STOCK` asset class — no new asset class needed), Flyway `V2__moomoo_bridge_support.sql` (3 new nullable `trades` columns: `external_source`/`external_entry_ref`/`external_exit_ref` + a partial unique index for dedup), matching `Trade` entity/`TradeRequest`/`TradeResponse`/`TradeMapper` changes, `TradeSpecifications`/`TradeController`/`TradeService` extended with `instrument` and `externalRef` list filters (used by the bridge for FIFO matching + dedup). Frontend TS types (`Trade`, `TradeRequest`) mirrored to match.
- `moomoo-bridge/` (new standalone Python project, `C:\PROJECTS\TRADEVAULT\moomoo-bridge\`): `bridge.py` (deal-push handler, FIFO matching, dedup, on-disk retry queue, rotating log), `tradevault_client.py` (REST client with auto-relogin on 401), `config.py` + `accounts.json` (config-driven watch list, v1 default = REAL MY account `286260077644734652` only) + `.env.example`.
- Confirmed real futu-api behavior while building: deal push payload columns are `trd_env, code, stock_name, deal_id, order_id, qty, price, trd_side, create_time, counter_broker_id, counter_broker_name, trd_market, status, jp_acc_type` — **no `acc_id`**, so watching 2+ accounts under the same security firm would misattribute deals (fine for v1's single-account scope, flagged in the spec for later).
- Known accepted limitation: closing a trade (`PUT`) always sends empty `tagIds`/`mistakeIds` since `TradeResponse` only returns tag/mistake *names*, not IDs — if Dejul manually tags a bridge-opened trade before it closes, the close-update will wipe those tags.

**Remaining manual steps to actually run it** (none of this can be done for Dejul — needs his own DB/UI/credentials):
1. `docker compose up -d --build` in TradeVault (picks up the V2 migration + new calculator)
2. In the TradeVault UI: create Account "moomoo MY (REAL)" (asset class `STOCK`) + its Calculation Profile (`calculator_key = STOCK_STANDARD`)
3. Copy that account's UUID into `moomoo-bridge/accounts.json` (replaces the placeholder)
4. Copy `.env.example` → `.env`, fill in TradeVault username/password
5. `pip install -r requirements.txt` (futu-api/requests/python-dotenv — requests + python-dotenv already installed this session; futu-api was installed earlier), then `python bridge.py`
6. Verify live: place a real trade via Claude Code + futu-api as usual, confirm the resulting row appears correctly in TradeVault

## Feature B: moomoo Order-Submission (EMAS Calculator + TP/SL) — Built, Not Yet Deployed
*Spec: `docs/specs/2026-08-25-moomoo-order-submission-design.md`. Designed + built 2026-08-26 (backend compiles, frontend builds, bridge syntax-checked). Depends on Feature A's `moomoo-bridge` process — extends it in place rather than duplicating it. Never run live yet.*

**Scope (v1)**: BUY entries only, REAL MY account only (same as Feature A). TradeVault becomes the one place order execution is triggered from, instead of ad-hoc Claude Code prompts. SELL-side flow, Equity-Based sizing, and the reverse "If Buy X Lots / If Buy RM X" calculators from Dejul's original spreadsheet are all explicitly deferred to v2 (his call).

**What was built:**
- **EMAS Calculator** (`frontend/src/lib/emasCalculator.ts` + `EmasCalculatorTable.tsx`) — Dejul's own risk-based position-sizing method, ported from his personal spreadsheet (confirmed via a screenshot + the exact cell formula he supplied, not guessed): Risk Value = Entry−SL, Max SL = Entry−ATR (red warning if SL exceeds 1×ATR), and for each R∈{1.00, 0.50, 0.25}: `Lots to Buy = ROUND(RiskAmount×R ÷ RiskValue, -2) ÷ 100` (his exact spreadsheet formula — round to nearest 100 shares, not round-down, which overrode an earlier round-down choice made before he shared the real formula), Capital Required, Size per Equity, and Stop Loss Value = RiskAmount×R (direct reference, not recomputed from the rounded lot count — matches his sheet). Pure client-side, recalculates live on every keystroke, no backend round-trip. Risk Amount auto-loads from `GET /api/trading-plan` (`maxRiskPerTrade`, existing endpoint), Current Equity from the selected `Account.currentBalance`.
- **New Order Ticket page** (`/orders/new`, `OrderTicketPage.tsx`, added to the sidebar nav) — Entry/SL/TP/ATR inputs, the EMAS table as reference, and a Quantity field that defaults to the R=1.00 row but stays freely editable (Dejul's call — no click-to-fill from the table).
- **Backend**: new `integration/moomoo` module (`MoomooOrderController` → `POST /api/moomoo/orders`, JWT-protected same as everything else) — a thin proxy only, holds no broker logic, forwards to the bridge via `RestTemplate` (new `tradevault.moomoo-bridge.base-url` config, default `http://localhost:8100`).
- **`moomoo-bridge` extended** (same process as Feature A, not a second one): `order_api.py` (new FastAPI server, runs in a background thread inside `bridge.py`, `/submit-order` endpoint places the entry LIMIT BUY only), `order_management.py` (places TP LIMIT SELL + SL STOP SELL automatically once the entry fill is observed via the *existing* deal-push handler — only for entries submitted through `/submit-order`, tracked via a new `pending_entries.json`; cancels the sibling TP/SL order when either fills, tracked via a new `oco_pairs.json`), `state_store.py` (small JSON-file key/value store backing both).
- Confirmed via `inspect.signature()` against the installed `futu-api`: exact `place_order`/`modify_order` parameter names (`aux_price` for STOP trigger, `ModifyOrderOp.CANCEL` for cancellation) — not guessed.
- **Accepted residual risk** (Dejul's explicit call): no broker-native OCO exists via OpenAPI (confirmed via Futu's own docs — only standalone order types, no position-linked TP/SL or OCO primitive). The self-built cancel-the-sibling approach has a real, unfixable race window — both legs can fill before the cancel call lands, over-closing the position — accepted rather than solved.
- **Accepted limitation**: if the entry order fills across multiple partial deals, only the *first* deal triggers TP/SL placement (the pending-entry record is popped on first match) — consistent with Feature A's own "v1 doesn't aggregate partial fills" stance.

**Remaining manual steps to actually run it** (on top of Feature A's deployment steps):
1. `pip install -r requirements.txt` in `moomoo-bridge` again (adds `fastapi` + `uvicorn`, already installed this session)
2. Everything else reuses Feature A's setup (same bridge process, same account, same `.env`) — nothing additional needed unless the bridge's HTTP port (default 8100) conflicts with something else on Dejul's machine
3. Testing plan (per spec §7, since this places real orders — no paper account for MY): first do a submit-only dry run (deliberately unfillable LIMIT price, confirm `SUBMITTED`, cancel manually) before ever letting a real entry fill and trigger auto TP/SL

## Session History (Last 5)

### 2026-08-26 - Designed + built Feature B (moomoo order-submission: EMAS Calculator + auto TP/SL + OCO)
- **Changes**: Brainstormed Feature B (parked in the previous session) via the brainstorming skill. Dejul shared his real personal position-sizing spreadsheet (Google Sheets, explicitly confidential — read via Google Drive access after he granted it, kept local to this project's own docs only, never published anywhere) to ground the "EMAS Calculator" instead of guessing the formula; corrected course twice when my own inferences were wrong (first assumed round-down lot sizing, then Dejul supplied the exact cell formula `ROUND(x,-2)/100`, which is round-to-nearest and overrode that earlier choice; also initially misread a garbled table extraction as 5 columns before Dejul corrected it to the real 4: Lots to Buy/Size per Equity/Capital Required/Stop Loss Value). Scope trimmed to Risk-Based sizing only for v1 (Equity-Based and the reverse "If Buy X/RM" calculators deferred) per Dejul's call. Wrote the approved design to `docs/specs/2026-08-25-moomoo-order-submission-design.md`, then built it in full: backend `integration/moomoo` proxy module (compiles clean), frontend EMAS Calculator + new Order Ticket page (builds clean), and extended the existing `moomoo-bridge` process (not a new one) with a FastAPI order-submission endpoint plus TP/SL auto-placement and self-built OCO cancel-on-fill logic — verified `place_order`/`modify_order`'s exact parameter names against the installed `futu-api` via `inspect.signature()` rather than assuming. All Python files syntax-checked. Neither Feature A nor B has been run against a live TradeVault/OpenD session yet — that's Dejul's next manual step.
- **Time Spent**: ~3 hours (estimated)

### 2026-08-25 - moomoo (Futu OpenAPI) trading integration: brainstorm, hands-on API validation, then built Feature A
- **Changes**: Followed the Panduan_Moomoo_Claude_Code_2026.pdf setup guide end-to-end on Dejul's machine: confirmed Python 3.12.10 + Claude Code already installed, installed `futu-api` 10.10.7008 via pip, found download link for OpenD GUI (Dejul installed + connected it himself), scanned all 7 security firms and identified FUTUMY as Dejul's real firm. Brainstormed (via brainstorming skill) a journal auto-sync bridge design (real-time push, FIFO matching, scope widened from SIMULATE-only to include the REAL MY account since the bridge never places orders). Mid-session Dejul described a second, bigger idea (Feature B, order-submission with TP/SL) — flagged as a separate subsystem per brainstorming-skill guidance and parked for its own future brainstorm; also clarified Feature A remains necessary even after Feature B exists, since it's the only way to catch positions closed manually in the moomoo app (as opposed to closes initiated from TradeVault itself). Hands-on validated real API behavior rather than continuing abstract design, since several assumptions turned out wrong when checked (MY quote support, MY paper account existence — the app's "Papertrade" MY tab turned out to be a separate gamified feature, not a real sub-account, STOP-in-paper support, native OCO). Wrote the approved design to `docs/specs/2026-08-25-moomoo-journal-sync-design.md`, then built Feature A: backend changes (`StockCalculator`, `V2` migration, `Trade`/DTO/spec/controller/service changes, frontend TS types) all compiling clean, plus the standalone `moomoo-bridge/` Python project (deal-push handling, FIFO matching, dedup, retry queue) — syntax-checked, all `futu-api` imports and the exact deal-push DataFrame columns verified against the installed package source (discovered it has no `acc_id` field — noted as a v1 limitation). Not yet run live — deployment needs Dejul's own manual steps (DB/Docker, UI account creation, credentials).
- **Time Spent**: ~4 hours (estimated)

### 2026-07-23/24 - Full build (all 11 deliverables) + live verification + debugging
- **Changes**: Built the entire app module-by-module per Dejul's spec, straight through without stopping: architecture doc, DB schema (16 tables), Spring Boot 3 backend (auth/JWT, accounts, asset classes, calculation profiles, Trade Engine, Calculator Engine with `ForexCalculator` + `CalculatorRegistry`, strategies, sessions, psychology, trading plan, dashboard + all charts, analytics, reports, portfolio, settings), React/TS/Vite/Tailwind/shadcn frontend (all pages, forms, charts), Docker Compose setup. Verified both halves actually compile (`mvn compile`/`package` and `npm run build` both clean) before considering it done.
  - Dejul then asked to actually run it. Found Docker Desktop running with existing SeQureMail containers already on ports 8080/8081 — remapped TradeVault to 8090 (backend)/8091 (frontend) to avoid collision.
  - **Bug #1** (found on first real boot): Hibernate schema validation failed — 9 tables (`asset_classes`, `trade_sessions`, `trade_calculation_results`, `trade_tags`, `trade_screenshots`, `trade_psychology`, `trade_mistakes`, `monthly_reports`, `settings`) were missing `created_at`/`updated_at` columns that every JPA entity requires via `BaseEntity`. Fixed in both the Flyway migration and the `docs/database-schema.sql` reference copy; wiped the (empty, just-created) db volume and rebuilt clean.
  - **Bug #2** (found when Dejul tried creating an Account from the UI): `LazyInitializationException` — `open-in-view: false` (deliberate choice) means every read method touching a lazy `@ManyToOne` needs its own `@Transactional(readOnly = true)`. Missing on 5 methods: `AccountService.listMine/getMine`, `CalculationProfileService.list`, `TradeService.list/get`, `PortfolioService.getPortfolio`, `TradePsychologyService.get`. Fixed, recompiled, rebuilt, and re-verified via curl (register → JWT → create account → list accounts → portfolio) — all green.
  - Set up Postgres access for Dejul: exposed db port 5433:5432 in compose (wasn't exposed at all before), gave HeidiSQL connection details. HeidiSQL connected but its Postgres tree view wouldn't populate tables (client-side quirk, not a data problem — confirmed data existed via API). Switched Dejul to DBeaver instead, which worked correctly and showed the live `accounts` data.
- **Time Spent**: ~5 hours (estimated)

## Historical Summary
[No history yet — this section is populated when session count exceeds 5]

## Technical Notes
- **Repository**: `C:\PROJECTS\TRADEVAULT\tradevault\` (no git repo initialized yet — filesystem only)
- **Key Dependencies**: Spring Boot 3, Java 21, Spring Security, JWT (no refresh token), Spring Data JPA, PostgreSQL, React 18, Vite, TailwindCSS, shadcn/ui, Recharts, React Hook Form, React Router, Docker/Docker Compose
- **Running locally**: `docker compose up -d --build` from the project root (needs `.env` — copy from `.env.example`, set `JWT_SECRET`). App: http://localhost:8091, API: http://localhost:8090/api, Postgres: `localhost:5433` (user/db `tradevault`, password in `.env` `DB_PASSWORD`)
- **DB client**: use DBeaver, not HeidiSQL — HeidiSQL connected fine but its object tree wouldn't populate Postgres tables for Dejul; DBeaver worked immediately
- **Core architectural rules** (must not be violated by future code):
  - Trade Engine = data storage only, zero calculation logic
  - Every asset-class calculator implements a common `TradeCalculator` interface — pluggable, no existing-code edits needed to add a new one
  - Forex math (R multiple, RR ratio, etc.) lives ONLY in Forex Calculator, never hardcoded in the Trade entity
  - UUID primary keys throughout; calculator-specific data stored in separate tables from generic `trades` table
  - Dark-mode-only, TradingView-inspired UI
  - Every entity extends `BaseEntity` (id/created_at/updated_at) — any new table needs both timestamp columns even if it has its own domain-specific timestamp too
  - `open-in-view` is `false` — any new read method that touches a lazy-loaded association must be `@Transactional(readOnly = true)` or it'll throw `LazyInitializationException`

---
**Last Updated**: 2026-08-26 | **Position**: #1/10 Active
