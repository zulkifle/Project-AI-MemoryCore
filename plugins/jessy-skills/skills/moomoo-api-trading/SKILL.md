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
| Account (REAL) | `acc_id = 286260077644734652` |
| Label | "moomoo MY (REAL)" |

**Bursa Malaysia execution IS supported via the API** — confirmed by Dejul's own moomoo-bridge already watching this REAL `MY`-market account for live deal pushes (`filter_trdmarket=TrdMarket.MY`). This overturns a naive assumption that moomoo only offers US/HK/SG/CN markets — `TrdMarket.MY` exists and works for Dejul's real account.

**Stock code format**: not yet independently re-derived here — reuse whatever code format flows through `moomoo-bridge`'s live deal pushes (`deal["code"]` in `bridge.py`) as ground truth. If placing an order for a new counter for the first time, pull a market snapshot first to confirm the code resolves correctly before submitting an order.

---

## Integration with position-size-calculator

1. Run [[position-size-calculator]] on Dejul's EP/SL/ATR/star input → get **Final Lots** (already floored to whole lots).
2. Order quantity (shares) = `Final Lots * 100` (Bursa board lot).
3. Order price = the calculator's Entry Price (C4), as a **limit** order (never market, per PDF golden rule).
4. Confirm stock code with Dejul if not already given alongside EP/SL/ATR.

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
```python
from futu import OpenSecTradeContext, SecurityFirm, TrdEnv, TrdMarket

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
