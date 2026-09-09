---
name: moomoo-api-trading
description: "MUST use when Dejul wants to execute, check, or place a stock order via the moomoo
             app/API (futu-api + OpenD) — 'execute guna moomoo', 'beli saham guna moomoo api',
             'check moomoo balance', 'buat order moomoo', or when combining
             [[position-size-calculator]] output into a real/paper order. Also use for moomoo
             account balance/position checks and market snapshots."
---

# moomoo-api-trading — Execute Orders via moomoo (futu-api + OpenD)
*Turn a [[position-size-calculator]] result straight into a moomoo order — paper or real*

## Origin
Derived from `G:\Other computers\WORK\GUIDE\BURSA\JEJAK LABUR\Panduan_Moomoo_Claude_Code_2026.pdf`
(Panduan Pemula 2026), cross-checked against Dejul's own working project
`C:\PROJECTS\TRADEVAULT\moomoo-bridge\` (Feature A/B, journal auto-sync + order submission).

## Activation
When this skill activates:
`📈 moomoo API Trading skill loaded — order execution via futu-api ready.`

---

## Architecture
```
Dejul types request → Claude Code writes Python → futu-api → OpenD (must be running, logged in) → moomoo servers
```
**OpenD MUST be open and logged in** (GUI version, status "Connected", port 11111) for anything to work. This cannot be automated by Claude Code — Dejul opens it manually before asking to execute.

## Prerequisites (one-time, already done per PDF)
- Python + `python -m pip install futu-api` installed
- OpenD (GUI version) installed, logged into Dejul's real moomoo account

---

## Confirmed Account Details (2026-09-09)

Verified directly from Dejul's own `moomoo-bridge/accounts.json` (TradeVault project, Feature A/B already live) — no need to re-run the Section 5 firm-scan from the PDF:

| Field | Value |
|---|---|
| Security Firm | `FUTUMY` |
| Market | `MY` (Bursa Malaysia — `TrdMarket.MY`) |
| Account (REAL) | `acc_id = 286260077644734652` (MARGIN) |
| Account (SIMULATE) | `acc_id = 5181982` (CASH — generic paper account, same id across all security firms, default balance RM/USD 1,000,000) |
| Label | "moomoo MY (REAL)" |

**Bursa Malaysia execution IS supported via the API** — confirmed by Dejul's own moomoo-bridge already watching this REAL `MY`-market account for live deal pushes (`filter_trdmarket=TrdMarket.MY`). This overturns a naive assumption that moomoo only offers US/HK/SG/CN markets — `TrdMarket.MY` exists and works for Dejul's real account.

**Stock code format — CONFIRMED live (2026-09-10)**: `MY.<Bursa numeric code>`, e.g. `MY.7233` (DUFU), `MY.0166` (INARI) — verified directly from Dejul's real open positions via `position_list_query`. Always pull a market snapshot first to double check a new/unfamiliar code before submitting an order.

---

## Confirmed Request Template (2026-09-10)

This is the exact format Dejul will paste — multi-line, one field per line:

```
execute
<CODE>
EP=<price>
SL=<price>
ATR=<value>
<star>★
Env : REAL              ← or omit / say SIMULATE for paper
```

Example:
```
execute
MY.5183
EP=8.50
SL=8.20
ATR=0.25
4★
Env : REAL
```

If `Env` is omitted, default to SIMULATE (Golden Rule #2 below still applies — never assume REAL).

## Integration with position-size-calculator

1. Run [[position-size-calculator]] on Dejul's EP/SL/ATR/star input → get **Final Lots** (already floored to whole lots).
2. Order quantity (shares) = `Final Lots * 100` (Bursa board lot).
3. Order price = the calculator's Entry Price (C4), as a **limit** order (never market, per PDF golden rule).
4. Confirm stock code with Dejul if not already given alongside EP/SL/ATR.

---

## Post-Entry Stop-Loss Handling (confirmed 2026-09-10)

Decision: after the BUY entry order is submitted, **check its fill status, and once FILLED, automatically submit a second, separate protective SELL order at the SL price** — in the same chat session (not a persistent background watcher; that's the deferred `moomoo-bridge` Feature B approach).

Flow:
1. Submit BUY entry (`place_order`, `trd_side=BUY`).
2. Poll status via `order_list_query` (filter by `order_id`) until `order_status` is `FILLED_ALL` (or a terminal non-fill state — report that back instead of silently retrying forever).
3. Once filled, submit the protective SELL order at SL price, same `code`, same `qty` as filled.

**Order types — confirmed by Dejul's own usual manual practice (2026-09-10)**:
- **Entry (BUY)** → **Limit Order** → `OrderType.NORMAL` (already what this skill uses).
- **SL (protective SELL)** → **Stop** → `OrderType.STOP` (trigger price = SL, via `aux_price`), NOT a plain limit sell. This matches Dejul's own convention in the moomoo app and resolves the earlier concern: a plain `OrderType.NORMAL` SELL at the SL price would fill immediately at the current bid instead of waiting for price to actually drop — `OrderType.STOP` is what avoids that.

```python
ret, data = trd_ctx.place_order(
    price=<SL_price>,           # reference/limit component
    aux_price=<SL_price>,       # trigger price for the stop
    qty=<filled_qty>,
    code='MY.<CODE>',
    trd_side=TrdSide.SELL,
    order_type=OrderType.STOP,
    trd_env=TrdEnv.SIMULATE,    # test here first; TrdEnv.REAL only after explicit confirmation
    acc_id=<acc_id>,
)
```

Still test once in SIMULATE the first time this runs end-to-end (check `ret == RET_OK`) before trusting it unattended on a REAL account — Dejul's manual-app experience confirms the order *type* is right, but the exact futu-api parameter shape (`aux_price` vs other trigger fields) hasn't been live-tested through this skill yet.

---

## 🔒 Golden Safety Rules (mandatory, every time)

1. **Snapshot before order** — always pull current market price first, show it to Dejul.
2. **SIMULATE (paper) by default** — only place a REAL order if Dejul explicitly says "REAL"/"sebenar"/"live" (or it's obviously implied by an ongoing REAL-trading conversation). Default `trd_env=TrdEnv.SIMULATE` otherwise.
3. **Read back before REAL execution** — for any REAL order, state code / qty / price / trd_env back to Dejul and get explicit confirmation before calling `place_order`. Treat this the same as any other hard-to-reverse action requiring confirmation.
4. **Respect a stated $ / RM cap** — if Dejul gives a max order value in the request, never submit above it. If no cap given for a REAL order, ask.
5. **Never expose the trading-unlock password** — it's needed for REAL orders (`unlock_trade`) but must never be typed into chat, saved to a script file, or logged. If unlocking is required, prompt Dejul to do it in the moomoo/OpenD app directly, or handle interactively without echoing it back.
6. **Verify after every attempt** — no error message ≠ order placed. Always check order status after submitting (`ret == RET_OK` and read back `order_status`).
7. **Limit orders only, price near market** — matches PDF guidance; avoid market orders and stale/far-off limit prices (`Price deviated too much` rejection).
8. **Whole-lot quantities only** — Bursa fractional shares aren't supported (`invalid literal for int()` if violated); this is already handled by position-size-calculator's floor rule.

---

## Reference Code Patterns

### Market snapshot (safe, read-only)
```python
from futu import OpenQuoteContext

quote_ctx = OpenQuoteContext(host='127.0.0.1', port=11111)
ret, data = quote_ctx.get_market_snapshot(['MY.<CODE>'])
quote_ctx.close()
```

### Balance / positions check (safe, read-only)

⚠️ **Confirmed quirk (2026-09-10)**: `filter_trdmarket=TrdMarket.MY` on the context breaks SIMULATE queries (`ERROR: the type of environment param is wrong`) — the SIMULATE paper account (`5181982`) isn't MY-market-specific. Drop the market filter for SIMULATE; keep it for REAL/`MY`.

⚠️ **Currency bug caught 2026-09-10 — DO NOT repeat**: `accinfo_query`'s top-level `total_assets`/`cash`/`market_val`/`power` fields are in the account's multi-currency **base display currency** (`HKD` for Dejul's REAL margin account, confirmed via the `currency` column), NOT automatically MYR — even though Dejul only trades the MY market. Reporting these fields to Dejul as "RM" without checking `currency` first gave a wrong figure (RM19,659 reported vs real RM10,201.59 in the app — a big overstatement). **Always read the `currency` field before labeling any amount.** For Dejul's account specifically, the correct MYR total is: `my_cash + sum(market_val of MY.* positions)` — his `hk_cash`/`us_cash` are both 0 (MY-only), confirmed live. Prefer the per-currency fields (`my_cash`, `my_avl_withdrawal_cash`, `hk_cash`, `us_cash`, ...) over the base-currency top-level fields whenever reporting a specific-currency figure to Dejul.

```python
from futu import OpenSecTradeContext, SecurityFirm, TrdEnv, TrdMarket

# SIMULATE — no filter_trdmarket
trd_ctx = OpenSecTradeContext(host='127.0.0.1', port=11111, security_firm=SecurityFirm.FUTUMY)
ret, data = trd_ctx.accinfo_query(trd_env=TrdEnv.SIMULATE, acc_id=5181982)
ret2, positions = trd_ctx.position_list_query(trd_env=TrdEnv.SIMULATE, acc_id=5181982)
trd_ctx.close()

# REAL — filter_trdmarket=MY is fine (and matches get_acc_list behavior)
trd_ctx = OpenSecTradeContext(host='127.0.0.1', port=11111,
                               security_firm=SecurityFirm.FUTUMY,
                               filter_trdmarket=TrdMarket.MY)
ret, data = trd_ctx.accinfo_query(trd_env=TrdEnv.REAL, acc_id=286260077644734652)
ret2, positions = trd_ctx.position_list_query(trd_env=TrdEnv.REAL, acc_id=286260077644734652)
trd_ctx.close()
```

### Place order (SIMULATE default — flip trd_env only after explicit confirmation)
```python
from futu import OpenSecTradeContext, SecurityFirm, TrdEnv, TrdMarket, TrdSide, OrderType, RET_OK

trd_ctx = OpenSecTradeContext(host='127.0.0.1', port=11111,
                               security_firm=SecurityFirm.FUTUMY,
                               filter_trdmarket=TrdMarket.MY)
ret, data = trd_ctx.place_order(
    price=<entry_price>,
    qty=<final_lots * 100>,
    code='MY.<CODE>',
    trd_side=TrdSide.BUY,
    order_type=OrderType.NORMAL,     # limit order
    trd_env=TrdEnv.SIMULATE,         # SIMULATE by default; TrdEnv.REAL only after explicit confirmation
    acc_id=286260077644734652,       # only relevant for the REAL account
)
trd_ctx.close()

if ret == RET_OK:
    order_id = data.iloc[0]['order_id']
    order_status = data.iloc[0]['order_status']
else:
    # inspect `data` (error message) and report to Dejul — do not retry blindly
    pass
```

---

## Troubleshooting (from PDF Section 10)

| Error | Cause | Fix |
|---|---|---|
| `Connection refused` / no account | OpenD closed or logged out | Tell Dejul to reopen OpenD, log in, wait for "Connected" |
| `Price deviated too much` | Limit price too far from market | Pull snapshot first, price near current market |
| `Insufficient buying power` | Not enough cash/margin | Reduce size or tell Dejul to top up |
| `invalid literal for int()` | Fractional shares | Use whole lots only (already handled by position-size-calculator) |
| Only SIMULATE account shows up | Wrong security firm | Should not happen — firm is confirmed as `FUTUMY` |

---

## Level History
- **Lv.1** — Base: derived from `Panduan_Moomoo_Claude_Code_2026.pdf`. Account details (FUTUMY, market=MY, REAL acc_id) confirmed from Dejul's own `moomoo-bridge` project rather than re-running the firm-scan — confirms Bursa Malaysia execution works via the API. Wired to consume [[position-size-calculator]] output directly (Final Lots × 100 = order qty). Golden safety rules encoded as mandatory steps (SIMULATE default, confirm before REAL, never expose unlock password, $ cap respected).
- **Lv.1.1** — Confirmed exact multi-line request template (code/EP/SL/ATR/star/Env) Dejul will paste. Explicitly confirmed this skill stays standalone (Claude Code + futu-api direct) and does NOT reuse `moomoo-bridge` — deferred per Dejul 2026-09-10. Added post-entry SL handling: check fill via `order_list_query`, then auto-submit a protective SELL order — flagged that a plain limit-sell at SL price is semantically wrong (fills immediately, not on price drop) and that `STOP`/`STOP_LIMIT` support for `TrdMarket.MY` is unverified — must test in SIMULATE before trusting on REAL.
- **Lv.1.2** — Live-tested SIMULATE portfolio check (2026-09-10): confirmed OpenD reachable, `get_acc_list()` per firm found SIMULATE acc_id `5181982` (CASH, default RM/USD 1,000,000 balance, no open positions) under every security firm including FUTUMY. Confirmed `filter_trdmarket=TrdMarket.MY` breaks SIMULATE queries — must omit the market filter for SIMULATE, keep it for REAL. Live-checked REAL portfolio too (2 open positions: MY.7233 DUFU, MY.0166 INARI) — confirmed real stock code format is `MY.<Bursa numeric code>`.
- **Lv.1.3** — Order type convention confirmed by Dejul (2026-09-10): Entry always Limit (`OrderType.NORMAL`), SL always Stop (`OrderType.STOP` with `aux_price`=trigger) — resolves the earlier open question about SL order semantics.
