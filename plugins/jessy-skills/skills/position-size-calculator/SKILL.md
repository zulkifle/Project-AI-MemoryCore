---
name: position-size-calculator
description: "MUST use when calculating stock position size, lot size, or risk-based position sizing
             for Bursa Malaysia (or any equity) trading. Trigger on 'position size calculator',
             'calculate lot saham', 'berapa lot nak beli', 'risk based position sizing',
             'position sizing saham', or when Dejul pastes just Entry Price / Stop Loss / ATR
             (+ optional star rating) to size a trade."
---

# position-size-calculator — Risk-Based Position Sizing (Bursa Malaysia)
*Calculate how many lots to buy based on % equity risk, not gut feel*

## Origin
Derived from Dejul's real Excel trading tool (2026-09-09):
`G:\Other computers\WORK\GUIDE\BURSA\EMAS\TRADING VIEW\position size calculator.txt`
Refined 2026-09-09 with quick-entry template, star-rating shortcut, floor-lot rule, and effective-SL rule.

## Activation
When this skill activates:
`📊 Position Size Calculator skill loaded — risk-based lot sizing ready.`

---

## Quick Entry (this is all Dejul needs to give each time)

```
Entry Price (C4) =
Stop Loss   (C5) =
ATR         (C6) =
Setup       = 5★ / 4★ / 3★
```

- **Current Equity (C2)** defaults to **RM10,000** — see [[project_trading_capital]] — unless Dejul states a different figure for that calculation.
- No need to re-explain the formulas each time — just plug values in and give the result table below.

### Star Rating → Risk (R) tier
| Setup | R |
|---|---|
| 5★ | 1 |
| 4★ | 0.5 |
| 3★ | 0.25 |

If Dejul doesn't give a star rating, show all three tiers (1R / 0.5R / 0.25R) like the original table.

---

## Full Inputs Reference

| Cell | Label | Meaning |
|---|---|---|
| C2 | Current Equity | Total trading capital (RM) — default RM10,000 |
| C4 | Entry Price | Planned buy price per share |
| C5 | Stop Loss | Manually chosen exit price if trade goes wrong |
| C6 | ATR | Average True Range (volatility measure, same unit as price) |

## Effective Stop Loss Rule (confirmed 2026-09-09)

The stop loss actually used in the risk math depends on price:

- **Entry Price (C4) ≥ RM1.00** → use the **ATR-based Max SL** as the effective stop: `EffectiveSL = F6 = C4 - C6`. (The manual C5 is still recorded/shown for reference, but is NOT used in the F4/lots math.)
- **Entry Price (C4) < RM1.00** → use the **manually given Stop Loss (C5)** directly: `EffectiveSL = C5`.

**Why this matters**: when EP ≥ RM1, `F4 = C4 - EffectiveSL = C4 - (C4 - C6) = C6` — i.e. **Risk Value collapses to exactly the ATR value**. Below RM1 (penny stocks), ATR-based stops are unreliable, so Dejul's manual SL judgment is used as-is.

## Derived Values (formulas)

| Cell | Label | Formula | Meaning |
|---|---|---|---|
| F2 | 1% Risk | `= 0.01 * C2` | RM amount risked at full (1R) position |
| F4 | Risk Value | `= C4 - EffectiveSL` (see rule above) | Risk per share, in RM |
| F5 | Stop Loss % | `= F4 / C4` | Stop distance as % of entry price |
| F6 | Max SL | `= C4 - C6` | ATR-based stop level (Entry − 1×ATR) |

---

## Lot Sizing (with floor rule)

Base 1R lots: `C16 = ROUND(F2/F4, -2) / 100` (rounds shares to nearest board lot of 100, then converts to lots — this is already a whole number).

For the chosen R tier (from star rating or table):

```
Raw Lots          = R * C16
Final Lots         = FLOOR(Raw Lots)              ← never round up, e.g. 2.25→2, 4.5→4
Capital Required   = Final Lots * C4 * 100
Actual Max Loss    = Final Lots * 100 * F4         ← recomputed from the FLOORED lots, not the raw target
```

**Why recompute Capital/Loss from Final (floored) Lots**: flooring means Dejul buys slightly less than the raw formula target, so the real capital used and real max loss are both slightly lower than `F2 * R` — show the actual numbers he'll really be trading, not the theoretical target.

---

## Worked Example (current rules)

Inputs: Equity = RM10,000 (default) | Entry = RM2.67 | Stop Loss (given) = RM2.56 | ATR = RM0.101 | Setup = 3★ (R=0.25)

```
Entry ≥ RM1 → EffectiveSL = F6 = 2.67 - 0.101 = 2.569  (manual SL 2.56 shown for reference only)
F2 (1% Risk)     = 0.01 * 10000           = RM100.00
F4 (Risk Value)  = 2.67 - 2.569           = RM0.101   (= ATR, as expected)
F5 (Stop Loss %) = 0.101 / 2.67           = 3.78%

C16 (base 1R lots) = ROUND(100/0.101, -2)/100 = ROUND(990.1, -2)/100 = 1000/100 = 10 lots

Setup 3★ → R = 0.25
Raw Lots   = 0.25 * 10 = 2.5
Final Lots = FLOOR(2.5) = 2 lots
Capital Required = 2 * 2.67 * 100 = RM534.00
Actual Max Loss   = 2 * 100 * 0.101 = RM20.20
```

Result to give Dejul: **buy 2 lots, RM534 capital, max loss ~RM20.20 if stopped out.**

### Same inputs, no star rating given → show all 3 tiers
| | 1R | 0.5R | 0.25R |
|---|---|---|---|
| Raw Lots | 10 | 5 | 2.5 |
| Final Lots (floored) | 10 | 5 | 2 |
| Capital Required | RM2,670.00 | RM1,335.00 | RM534.00 |
| Actual Max Loss | RM101.00 | RM50.50 | RM20.20 |

---

## Output Format
Keep it short — a small table (or one line if only one tier requested):
`Entry / EffectiveSL used / Final Lots / Capital Required / Actual Max Loss / % equity risked`

---

## Implementation Notes
- Bursa Malaysia: 1 lot = 100 shares.
- `C16` (base 1R lots) is already a round number of lots (from `ROUND(...,-2)/100`); fractional lots only appear at 0.5R/0.25R tiers — always FLOOR, never round up.
- Guard against `F4 <= 0` — invalid input (SL on the wrong side of Entry for a long trade).
- Assumes a **long** position. For a short, flip the subtraction directions throughout.
- If Entry < RM1, F6/Max SL is still computable but is NOT used as the effective stop — only informational.

---

## Level History
- **Lv.1** — Base: Formulas confirmed directly by Dejul (2026-09-09) from his real Excel tool. Worked example added for verification.
- **Lv.2** — Quick-entry template (EP/SL/ATR + star rating only, equity defaults to RM10,000), star→R mapping (5★=1R/4★=0.5R/3★=0.25R), floor-lot rule (never round up), and effective-SL rule (≥RM1 uses ATR-based Max SL, <RM1 uses manual SL) — all confirmed by Dejul 2026-09-09.
