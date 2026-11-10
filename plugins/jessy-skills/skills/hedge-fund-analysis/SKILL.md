---
name: hedge-fund-analysis
description: "MUST use when Dejul asks for a comprehensive analysis of a stock, company or commodity
             in the style of a professional equity analyst / hedge fund portfolio manager. Trigger on
             'analisis saham [X]', 'analyse [ticker]', 'equity analysis', 'hedge fund analysis',
             'fundamental analysis', 'buat analisis [company]', 'fair value [stock]', 'bull base bear',
             'analisis FCPO', 'analisis komoditi', or when Dejul pastes the 'Act as a professional
             equity analyst' prompt. Covers Bursa Malaysia, US stocks and commodity futures (FCPO)."
---

# hedge-fund-analysis — Equity Analyst / Hedge Fund PM Framework
*One fixed structure for every stock or commodity deep-dive: view → business → numbers → drivers → risks → valuation → technicals → scenarios → plan → thesis*

## Origin
Dejul's own analysis prompt, given 2026-11-10 ("remember this skill"). The same framework,
adapted for futures, drives the FCPO Morning Brief in the TradeVault `position-sizing-bot`
(`fcpo_brief.py`, `SYSTEM_PROMPT`).

## Activation
When this skill activates:
`📈 Hedge Fund Analysis skill loaded — full analyst framework ready.`

---

## Golden Rules (apply to every section)
1. **Latest verified information only** — use web search; use the latest available financial results (quarterly/annual report, Bursa announcement, 10-Q/10-K).
2. **State the date and the share price used** at the top of the report.
3. **Separate verified facts from assumptions** — every verified figure carries its source + date, e.g. `(Q3 FY26 report, 26 Nov 2026)`; every estimate/projection/judgement starts with **"Andaian:"** / **"Assumption:"**.
4. **Never invent numbers.** If a figure can't be verified or sources disagree, say so.
5. **Preferred sources:** Bursa Malaysia announcements, company annual/quarterly reports, SEC filings, Reuters, Bloomberg, The Edge, The Star, Bernama, i3investor (data only), Yahoo Finance / Stockanalysis for multiples. Commodities: MPOB, MPOC, ITS/AmSpec, GAPKI, USDA, Oil World.
6. **Language:** reply in casual Malay with English finance terms (Dejul's chat style); written files in English.
7. General market analysis — not personal advice. For position size, hand off to [[position-size-calculator]] (equity RM10,000 — see [[project_trading_capital]]); for execution, [[moomoo-api-trading]].

---

## Report Structure — Stocks

| # | Section | Must cover |
|---|---|---|
| 1 | **Executive Summary** | Clear investment view: **Positive / Neutral / Negative**; date + share price used; 3–5 key reasons |
| 2 | **Business Overview** | Business model, revenue sources, competitive advantages (moat), major operating segments |
| 3 | **Financial Performance** | Revenue, net profit, margins (gross/EBITDA/net), EPS, cash flow (OCF, FCF), balance sheet, debt (net gearing), important multi-quarter/year trends |
| 4 | **Growth Drivers & Catalysts** | Factors that could raise revenue, earnings or valuation (contracts, capacity, M&A, policy, rate cuts, index inclusion) |
| 5 | **Key Risks** | Operational, financial, industry, currency, regulatory, valuation |
| 6 | **Valuation** | P/E, P/B, EV/EBITDA, dividend yield, historical valuation (vs own 5-yr average), peer comparison table |
| 7 | **Technical Analysis** | Current trend, support, resistance, breakout levels, momentum (RSI/MACD), trading volume |
| 8 | **Bull / Base / Bear Cases** | Per scenario: assumptions, estimated earnings or EPS, fair valuation multiple, estimated fair value (+ rough probability) |
| 9 | **Trading / Investment Plan** | Potential accumulation zones, breakout confirmation levels, invalidation levels, catalysts to monitor (results date, ex-date, AGM, policy events) |
| 10 | **Final Investment Thesis** | Why an investor should consider buying, holding or avoiding the stock |

Fair value per scenario = estimated EPS × fair P/E (or BVPS × fair P/B for banks/REITs-style, or DPS ÷ target yield for dividend plays) — show the arithmetic.

---

## Report Structure — Commodity futures (FCPO and similar)
A futures contract has no EPS, P/E or balance sheet. Keep the same 10 sections, swap the content:

| Equity section | Commodity equivalent |
|---|---|
| Business Overview | **Market overview** — contract spec (FCPO: 25 t, RM/t, 3rd month = benchmark), main producers (MY, ID), main demand (India, China, EU, Pakistan, biodiesel), edge vs rival oils |
| Financial Performance | **Fundamentals** — MPOB production / exports / imports / closing stocks (m/m, y/y, 3–6 month trend); ITS/AmSpec month-to-date exports; GAPKI, Indonesia reference price, levy, B40/B50 |
| Valuation (P/E, P/B, EV/EBITDA, yield) | **Relative valuation** — palm vs soyoil spread, POGO spread, price vs 1-yr range and historical average, forward curve (contango/backwardation, Dec/Jan/3rd-month spreads), peers (Dalian olein, CBOT soyoil, sunflower), price vs cost of production → cheap / fair / expensive |
| Bull/Base/Bear EPS & multiple | **Price range in RM/t** for the benchmark over 1–3 months + assumptions + probability (sum 100%) |
| Executive Summary | Also: yesterday's Dec/Jan settlements + night session, overnight Dalian/CBOT/crude/ringgit, and the rule-based scorecard bias (see below) |

### FCPO scorecard + bias (keep whenever FCPO is analysed)
| Factor | +1 / −1 rule |
|---|---|
| FCPO trend | last 3rd-month settlement vs ~10-day average (MPOC) |
| CBOT soyoil | ±0.3% overnight |
| Brent | ±0.5% overnight |
| USD/MYR | ±0.15% (weaker ringgit = +1) |
| Dalian | from news (−1/0/+1) |
| Exports | from ITS/AmSpec news (−1/0/+1) |

Total ≥ +2 = 🟢 **BULLISH**, ≤ −2 = 🔴 **BEARISH**, else 🟡 **NEUTRAL**. Compare the analyst view (Positive/Neutral/Negative) with the scorecard bias and explain any gap. Flag if a major release (MPOB ~10th, exports 10th/15th/20th/25th/month-end) lands today — it can flip the bias. FCPO sessions: 10:30–12:30, 14:30–18:00 MYT + T+1 night session.

---

## Output Format (chat)
- Header line: `📊 [NAME / TICKER] — [date] — Harga: RMx.xx (source, time)`
- Executive Summary first, in a short block with the view in bold.
- Tables for financials, valuation/peers and the bull/base/bear scenarios.
- End with a 2–4 line Final Thesis and a sources line.
- For Telegram/plain text (the bot), no Markdown tables — sections with emoji headings and "-" bullets.
- **TL;DR mode** (Dejul prefers short — 2026-11-10: "short kan sahaja, tldr"): same sections, 1–3 lines each (key number + a short why), ~2,200–2,800 chars, must fit ONE Telegram message — "jangan terlampau ringkas, sedikit penerangan, 1 mesej sahaja". The FCPO bot always uses this mode; use the full report in chat only when Dejul asks for a full/deep analysis.

---

## Level History
- **Lv.1** — Base (2026-11-10): Dejul's equity analyst / hedge fund PM prompt, plus the FCPO commodity adaptation and scorecard bias from the TradeVault FCPO Morning Brief (image `customsupp:1.4`); bot prompt trimmed to TL;DR mode the same day.
