# TradeVault - Self-Hosted Personal Trading OS
*"One Platform. Every Trade. Every Market." — Modular monolith trading journal/analytics platform for Dejul's personal use.*

## Project Overview
- **Type**: Web App
- **Client**: Internal (personal use)
- **Period**: 2026-07-22 - Active
- **Tech Stack**: Backend: Spring Boot 3 + Java 21 + Spring Security (JWT) | Frontend: React + TypeScript + Vite + TailwindCSS + shadcn/ui + React Router + React Hook Form + Recharts | Database: PostgreSQL (Spring Data JPA)
- **Completion**: 85%
- **Duration**: ~5 hours (estimated — no Auto-Commit time tracking; TradeVault has no git repo of its own yet)
- **Due Date**: TBD

## Current Status
- **Last Session**: 2026-07-23/24 - Full application built module-by-module and verified running live in Docker (backend + frontend + Postgres), including finding and fixing two real runtime bugs
- **Next Steps**: 1) Continue real usage — log actual trades, verify charts/reports/analytics against real data 2) Consider `git init` for the TradeVault app repo itself (separate from Jessy's tracking repo) 3) Add `frontend/eslint.config.js` (the `lint` npm script currently has no config to run against) 4) V2 scope, when ready: add calculators for Bursa/Crypto/Futures/etc. per the pluggable `TradeCalculator` pattern — no Trade Engine changes needed
- **Known Issues**: None currently open. `smoketest` account + one `XM Demo` (FOREX) account exist in the dev DB from verification testing — harmless, safe to delete or ignore.

## Session History (Last 5)

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
**Last Updated**: 2026-07-24 | **Position**: #1/10 Active
