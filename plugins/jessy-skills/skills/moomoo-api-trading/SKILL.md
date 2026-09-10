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

## 📋 Prompt Templates (Quick Reference)

All confirmed working 2026-09-10 unless marked otherwise.

**Check total assets / portfolio — REAL**
```
check portfolio real
```
(or "check total assets real", "check balance real" — any phrasing works, all read-only/safe)

**Check total assets / portfolio — SIMULATE**
```
check portfolio simulate
```

**Market snapshot (price check, read-only)**
```
snapshot MY.<CODE>
```

**Position sizing only (no order placed)** — see [[position-size-calculator]]
```
EP=
SL=
ATR=
<star>★          ← optional, omit to see all 3 R tiers
```

**Execute order — SIMULATE (paper, default)**
```
execute
<CODE>
EP=
SL=
ATR=
<star>★
```
(no `Env` line = SIMULATE)

**Execute order — REAL**
```
execute
<CODE>
EP=
SL=
ATR=
<star>★
Env : REAL
```
Triggers full flow: position size → snapshot → readback + confirm → place BUY limit → poll for fill → place SL stop order.

**Check open orders**
```
check open orders real
```
or `check open orders simulate`

**Cancel an order** — ⚠️ not yet built/tested this session; ask if needed and it'll be added then verified in SIMULATE first.

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
    price=0,                    # unused for a pure STOP order — leave 0
    aux_price=<SL_price>,       # trigger price — this is what actually matters
    qty=<filled_qty>,
    code='MY.<CODE>',
    trd_side=TrdSide.SELL,
    order_type=OrderType.STOP,
    trd_env=TrdEnv.SIMULATE,    # test here first; TrdEnv.REAL only after explicit confirmation
    acc_id=<acc_id>,
)
```

**Confirmed live 2026-09-10** by reading Dejul's own real pending orders (`order_list_query`) — his existing manually-placed bracket orders (DUFU, INARI) show exactly this shape: `order_type=STOP`, `price=0.00`, real trigger in `aux_price` (e.g. DUFU SL trigger `aux_price=2.71`). His TP orders use `order_type=NORMAL` with `price` set normally (no `aux_price`). This is ground truth, not just the manual-app convention — the parameter shapes above are now verified.

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

# SIMULATE — NO filter_trdmarket (breaks queries on generic paper account)
trd_ctx = OpenSecTradeContext(host='127.0.0.1', port=11111, security_firm=SecurityFirm.FUTUMY)
ret, data = trd_ctx.accinfo_query(trd_env=TrdEnv.SIMULATE, acc_id=5181982)
if ret == RET_OK:
    # ALWAYS check currency before reporting amounts!
    currency = data.iloc[0]['currency']  # e.g. 'HKD', 'MYR', 'USD'
    if currency == 'MYR':
        my_cash = data.iloc[0]['my_cash']  # Only for MYR specifically
    else:
        # For multi-currency, use per-currency fields
        my_cash = data.iloc[0]['my_cash']  # might be 0
ret2, positions = trd_ctx.position_list_query(trd_env=TrdEnv.SIMULATE, acc_id=5181982)
trd_ctx.close()

# REAL — WITH filter_trdmarket=MY (Dejul's account is MY-only)
trd_ctx = OpenSecTradeContext(host='127.0.0.1', port=11111,
                               security_firm=SecurityFirm.FUTUMY,
                               filter_trdmarket=TrdMarket.MY)
ret, data = trd_ctx.accinfo_query(trd_env=TrdEnv.REAL, acc_id=286260077644734652)
if ret == RET_OK:
    currency = data.iloc[0]['currency']  # expect 'HKD' for Dejul
    my_cash = data.iloc[0]['my_cash']    # actual MYR available
ret2, positions = trd_ctx.position_list_query(trd_env=TrdEnv.REAL, acc_id=286260077644734652)
trd_ctx.close()
```

### Complete order execution flow (snapshot → size → place → poll → SL)

**Full working template — ready to run, includes error handling & SL auto-placement:**

```python
import time
from futu import (
    OpenSecTradeContext, OpenQuoteContext, SecurityFirm, TrdEnv, TrdMarket,
    TrdSide, OrderType, RET_OK
)

CODE = 'MY.5199'
ENTRY_PRICE = 2.28
STOP_LOSS = 2.25
QUANTITY = 300  # Already sized via position-size-calculator
TRD_ENV = TrdEnv.REAL  # Flip to TrdEnv.SIMULATE to test first
ACC_ID = 286260077644734652 if TRD_ENV == TrdEnv.REAL else 5181982

print("=" * 70)
print(f"🚀 Order: {CODE} x{QUANTITY} @ RM{ENTRY_PRICE} | SL RM{STOP_LOSS} | {TRD_ENV.name}")
print("=" * 70)

try:
    # [1] SNAPSHOT — confirm market price is near our limit
    print(f"\n[1] Market snapshot for {CODE}...")
    quote_ctx = OpenQuoteContext(host='127.0.0.1', port=11111)
    ret, data = quote_ctx.get_market_snapshot([CODE])
    quote_ctx.close()
    
    if ret != RET_OK:
        print(f"✗ Snapshot failed: {data}")
        raise Exception("Cannot verify market price")
    
    current_price = data.iloc[0]['last_price']
    print(f"✓ Current price: RM{current_price:.2f}")
    
    if abs(current_price - ENTRY_PRICE) > 0.5:  # safety check: >50 sen away = suspicious
        print(f"⚠️  WARNING: Entry RM{ENTRY_PRICE} is RM{abs(current_price - ENTRY_PRICE):.2f} away from market RM{current_price:.2f}")
        print(f"   Risk: order may be rejected with 'Price deviated too much'")
    
    # [2] PLACE ENTRY ORDER
    print(f"\n[2] Placing BUY entry order...")
    trd_ctx = OpenSecTradeContext(host='127.0.0.1', port=11111,
                                  security_firm=SecurityFirm.FUTUMY,
                                  filter_trdmarket=TrdMarket.MY if TRD_ENV == TrdEnv.REAL else None)
    # Note: filter_trdmarket ONLY for REAL; omit for SIMULATE
    
    ret, data = trd_ctx.place_order(
        price=ENTRY_PRICE,
        qty=QUANTITY,
        code=CODE,
        trd_side=TrdSide.BUY,
        order_type=OrderType.NORMAL,
        trd_env=TRD_ENV,
        acc_id=ACC_ID,
    )
    
    if ret != RET_OK:
        trd_ctx.close()
        print(f"✗ Order rejected: {data}")
        raise Exception(f"Order placement failed: {data}")
    
    order_id = str(data.iloc[0]['order_id'])
    order_status = str(data.iloc[0]['order_status'])
    print(f"✓ Entry submitted: Order ID {order_id} | Status {order_status}")
    
    # [3] POLL FOR FILL (with timeout)
    print(f"\n[3] Polling for fill (max 30s)...")
    timeout = time.time() + 30
    filled_qty = 0
    
    while time.time() < timeout:
        ret, data = trd_ctx.order_list_query(trd_env=TRD_ENV, acc_id=ACC_ID, order_id=order_id)
        if ret == RET_OK and not data.empty:
            order_status = str(data.iloc[0]['order_status'])
            filled_qty = data.iloc[0]['dealt_qty']
            
            if order_status == 'FILLED_ALL':
                print(f"✓ FILLED: {filled_qty} shares")
                break
            elif order_status in ['CANCELLED_ALL', 'CANCELLED_PART']:
                print(f"⚠️  Order cancelled: {order_status}")
                trd_ctx.close()
                raise Exception(f"Order was cancelled: {order_status}")
            else:
                print(f"  Status: {order_status} (filled: {filled_qty}/{QUANTITY})")
        
        time.sleep(1)
    
    if filled_qty == 0:
        print(f"✗ Order did not fill within 30s — still pending. Check moomoo app.")
        trd_ctx.close()
        raise Exception("Order fill timeout")
    
    # [4] AUTO-PLACE SL STOP ORDER
    print(f"\n[4] Placing protective SL order...")
    ret, sl_data = trd_ctx.place_order(
        price=0,                   # unused for STOP order
        aux_price=STOP_LOSS,       # trigger price
        qty=filled_qty,
        code=CODE,
        trd_side=TrdSide.SELL,
        order_type=OrderType.STOP,
        trd_env=TRD_ENV,
        acc_id=ACC_ID,
    )
    
    trd_ctx.close()
    
    if ret != RET_OK:
        print(f"✗ SL order rejected: {sl_data}")
        print(f"⚠️  Entry filled but SL FAILED — manually place SELL STOP @ RM{STOP_LOSS} in moomoo app")
        raise Exception(f"SL placement failed: {sl_data}")
    
    sl_order_id = str(sl_data.iloc[0]['order_id'])
    print(f"✓ SL placed: Order ID {sl_order_id} | Trigger RM{STOP_LOSS}")
    
    # SUCCESS
    print("\n" + "=" * 70)
    print("✅ ORDER EXECUTED SUCCESSFULLY")
    print("=" * 70)
    print(f"Entry Order ID:  {order_id}")
    print(f"Filled:          {filled_qty} shares @ RM{ENTRY_PRICE}")
    print(f"Capital:         RM{filled_qty * ENTRY_PRICE:.2f}")
    print(f"\nSL Order ID:     {sl_order_id}")
    print(f"Stop trigger:    RM{STOP_LOSS}")
    print(f"Max loss:        RM{(ENTRY_PRICE - STOP_LOSS) * filled_qty:.2f}")
    print("=" * 70)

except Exception as e:
    print(f"\n❌ ERROR: {e}")
    import traceback
    traceback.print_exc()
```

---

## Troubleshooting (from PDF Section 10)

| Error | Cause | Fix |
|---|---|---|
| `Connection refused` / `socket.gaierror` | OpenD closed or logged out | Tell Dejul: "Buka OpenD, login, tunggu 'Connected' status" |
| `Price deviated too much` | Limit price RM0.5+ away from market | Pull snapshot first (built into flow now), adjust limit if market moved |
| `Insufficient buying power` | Not enough cash/margin for qty | Reduce quantity or ask Dejul to top up account |
| `invalid literal for int()` | Fractional shares submitted | Use whole lots only — position-size-calculator floors already |
| Order filled but SL fails | Market moved or insufficient power | SL check in script now; catch error, manual placement fallback |
| `filter_trdmarket` breaks SIMULATE | Filter only works on real/MY markets | Script now: `filter_trdmarket=TrdMarket.MY if TRD_ENV == TrdEnv.REAL else None` |
| Wrong currency reported | Didn't check `currency` field first | Script now: read `accinfo_query`'s `currency` before labeling RM/HKD/USD |
| Order hangs / no fill in 30s | Network lag or market not moving | Script timeout now 30s, report back to Dejul, suggest manual check |

---

## Level History
- **Lv.1** — Base: derived from `Panduan_Moomoo_Claude_Code_2026.pdf`. Account details (FUTUMY, market=MY, REAL acc_id) confirmed from Dejul's own `moomoo-bridge` project rather than re-running the firm-scan — confirms Bursa Malaysia execution works via the API. Wired to consume [[position-size-calculator]] output directly (Final Lots × 100 = order qty). Golden safety rules encoded as mandatory steps (SIMULATE default, confirm before REAL, never expose unlock password, $ cap respected).
- **Lv.1.1** — Confirmed exact multi-line request template (code/EP/SL/ATR/star/Env) Dejul will paste. Explicitly confirmed this skill stays standalone (Claude Code + futu-api direct) and does NOT reuse `moomoo-bridge` — deferred per Dejul 2026-09-10. Added post-entry SL handling: check fill via `order_list_query`, then auto-submit a protective SELL order — flagged that a plain limit-sell at SL price is semantically wrong (fills immediately, not on price drop) and that `STOP`/`STOP_LIMIT` support for `TrdMarket.MY` is unverified — must test in SIMULATE before trusting on REAL.
- **Lv.1.2** — Live-tested SIMULATE portfolio check (2026-09-10): confirmed OpenD reachable, `get_acc_list()` per firm found SIMULATE acc_id `5181982` (CASH, default RM/USD 1,000,000 balance, no open positions) under every security firm including FUTUMY. Confirmed `filter_trdmarket=TrdMarket.MY` breaks SIMULATE queries — must omit the market filter for SIMULATE, keep it for REAL. Live-checked REAL portfolio too (2 open positions: MY.7233 DUFU, MY.0166 INARI) — confirmed real stock code format is `MY.<Bursa numeric code>`.
- **Lv.1.3** — Order type convention confirmed by Dejul (2026-09-10): Entry always Limit (`OrderType.NORMAL`), SL always Stop (`OrderType.STOP` with `aux_price`=trigger) — resolves the earlier open question about SL order semantics.
- **Lv.1.4 — Bug fixes (2026-09-10, post-manual-order)** — Fixed 5 critical bugs preventing auto-execution:
  1. **SIMULATE filter bug**: `filter_trdmarket=TrdMarket.MY` breaks SIMULATE queries. Now: conditional — only apply filter for REAL orders, omit for SIMULATE.
  2. **Currency reporting bug**: Reported RM19,659 without checking `currency` field (was actually HKD). Now: always read `currency` before labeling amounts; prefer per-currency fields (`my_cash`, `hk_cash`, etc.) over base fields.
  3. **No complete execution flow**: Manual Python debugging required each time. Now: full ready-to-run script template (snapshot → place → poll → SL) with error handling baked in.
  4. **SL placement unclear**: Loop logic and timeout not specified. Now: explicit 30s timeout with clear status reporting.
  5. **Poor error messages**: "inspect data" left user guessing. Now: specific error messages per failure mode (connection/price/power/fill/SL) with actionable fixes.
