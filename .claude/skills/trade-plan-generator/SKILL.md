---
name: trade-plan-generator
description: Generates a complete swing trade plan for any ticker — stocks use Minervini SEPA, ETFs use the ETF Swing Trading strategies (Mean Reversion, MA Crossover, RSI-based, Volatility-based). Always produces a structured plan with entry, stop, and profit targets — and clearly flags any rule violations. Saves every plan as a .md file to the Trade Plans folder. Use this skill whenever the user mentions a specific ticker and wants a trade plan, setup analysis, entry/stop/target levels, risk sizing, or position sizing. Also trigger when the user asks things like "is X a good setup?", "plan a trade on X", "what's my risk on X?", or "walk me through the trade on X". If a ticker symbol appears alongside any trading intent, use this skill.
---

# Trade Plan Generator

Generate a complete, honest trade plan for any ticker. Automatically routes to the correct strategy based on instrument type, then saves the plan as a dated .md file.

---

## Step 0 — Route to the Correct Strategy

Before doing anything else, determine whether the ticker is an **individual stock** or an **ETF**.

### Known ETF Tickers (non-exhaustive)

**Index ETFs**: SPY, QQQ, IWM, DIA, VOO, VTI, IVV, RSP, MDY, VB, ONEQ
**Leveraged/Inverse**: TQQQ, SQQQ, SPXL, SPXS, UPRO, TNA, TZA, QLD, SSO, UDOW, SDOW, LABU, LABD, SOXL, SOXS, FNGU, FNGD, TECL, TECS
**Sector ETFs**: XLK, XLF, XLE, XLV, XLI, XLC, XLRE, XLP, XLU, XLB, XLY, SMH, IBB, KRE, XOP, GDX, GDXJ, XBI, ARKK, ARKG
**International**: EEM, EFA, FXI, VWO, INDA, EWZ, EWJ
**Commodity**: GLD, SLV, USO, UNG, DBA
**Bond**: TLT, IEF, SHY, HYG, LQD, AGG, BND

### Routing Rules

```
IF ticker is in the ETF list above OR ends in common ETF patterns:
   → Use the ETF Swing Trading skill (etf-swing-trading)
   → Run the auto-selection logic to pick the best of the 4 strategies
   → Let the user override if they want a different strategy

IF ticker is an individual stock:
   → Use Minervini SEPA framework (below)

IF uncertain:
   → Check via Massive MCP if the ticker is an ETF or stock
   → Ask the user if still ambiguous
```

When routing to the ETF skill, follow ALL instructions in the `etf-swing-trading` SKILL.md for strategy selection, analysis, and output format.

When routing to Minervini SEPA, continue with Steps 1-5 below.

---

## Step 1 — Fetch Data via Massive MCP (Stocks — SEPA)

Pull at least 252 trading days of OHLCV data. Use the `mcp__Massive_Market_Data__query_data` or `mcp__Massive_Market_Data__call_api` tools. You need:

- Daily OHLCV (close, high, low, volume) — minimum 252 bars
- Current price and today's volume
- 50-day average volume (for breakout volume confirmation)

If Massive MCP is unavailable, say so and ask the user to paste in price data.

---

## Step 2 — Run the SEPA Checklist

Work through each gate in order. Record pass/fail for each item — you'll need this for the violations section.

### Trend Template (all 8 must pass for Stage 2)

Calculate from the fetched data:

| # | Criterion | Formula |
|---|-----------|---------|
| 1 | Price > 150-day MA | `close > MA150` |
| 2 | Price > 200-day MA | `close > MA200` |
| 3 | 150-day MA > 200-day MA | `MA150 > MA200` |
| 4 | 200-day MA trending up >=1 month | `MA200_now > MA200_22days_ago` |
| 5 | 50-day MA > 150-day MA AND > 200-day MA | `MA50 > MA150 AND MA50 > MA200` |
| 6 | Price > 50-day MA | `close > MA50` |
| 7 | Price >= 25% above 52-week low | `close >= low_52w * 1.25` |
| 8 | Price within 25% of 52-week high | `close >= high_52w * 0.75` |

### VCP Assessment

Look at the last 60 trading days for a base structure. Check:
- At least 2 contractions (swing high to swing low sequences)
- Each contraction shallower than the last
- Volume declining through the base (supply drying up)
- A clear pivot point (high of the last contraction)

This doesn't need to be a perfect algorithmic detection — use your judgment on the chart structure based on the price data. Flag if the base looks wide-and-loose vs. tight.

### Entry Zone

- **Pivot**: High of the last contraction
- **Buy zone**: Pivot to Pivot x 1.05
- **Extended**: Current price > Pivot x 1.05 -> do NOT recommend entry
- **Not yet**: Current price < Pivot -> note "awaiting breakout"

---

## Step 3 — Calculate the Trade Plan

Use these inputs to build the plan. If you don't know the user's account size, assume $100,000 and note this assumption.

**Entry price**: Use current price if in buy zone; use pivot price for planning if "awaiting breakout"

**Stop loss**:
- Preferred: 1% below the low of the last contraction (structure-based)
- Fallback: Entry x 0.93 (7% fixed)
- Hard max: 8% below entry — if structure-based stop is wider than 8%, use 8%

**Position size**:
- Risk per trade: 1% of account = `account x 0.01`
- Shares: `risk_dollars / (entry - stop)`
- Cap at 25% of account value

**Profit targets**:
- 2R target: `entry + (entry - stop) x 2`
- 3R target: `entry + (entry - stop) x 3`
- 20% target: `entry x 1.20`

**Breakout volume needed**: 50-day avg volume x 1.5

---

## Step 4 — Output Format (Stocks — SEPA)

Always use this exact layout. Fill in every field — use "N/A" or "Unable to calculate" if data is missing.

```
=============================================
  TRADE PLAN -- [TICKER] ([Company])  [DATE]
=============================================

  SEPA STATUS: [FULL PASS | PARTIAL | STAGE 2 FAIL]

[Only show this block if any criteria failed:]
  VIOLATIONS
--------------
* [Criterion name]: [Brief explanation of what failed and why it matters]
* [Add one bullet per violation]

[If stock is Extended:]
  EXTENDED -- Current price is [X]% above pivot. Do not enter.
  Wait for a new base to form before considering a trade.

----------------------------------------------
  TREND TEMPLATE
----------------------------------------------
  Price vs 50 MA:    $[price] vs $[ma50]      [PASS/FAIL]
  Price vs 150 MA:   $[price] vs $[ma150]     [PASS/FAIL]
  Price vs 200 MA:   $[price] vs $[ma200]     [PASS/FAIL]
  150 MA vs 200 MA:  $[ma150] vs $[ma200]     [PASS/FAIL]
  200 MA trending:   [Up/Down/Flat]            [PASS/FAIL]
  52w High:          $[high]  ([X]% from price)[PASS/FAIL]
  52w Low:           $[low]   ([X]% from price)[PASS/FAIL]
  Stage:             [Stage 1 / 2 / 3 / 4]

----------------------------------------------
  VCP SETUP
----------------------------------------------
  Base length:       [X] weeks
  Contractions:      [N] contractions -- [tightening/widening]
  Volume dry-up:     [Yes/No/Partial]
  Tightness:         [Tight <8% | Moderate 8-15% | Wide >15%]
  Pivot:             $[pivot]

----------------------------------------------
  ENTRY & RISK
----------------------------------------------
  Current price:     $[price]
  Entry zone:        $[pivot] -- $[pivot x 1.05]
  Status:            [In zone / Awaiting breakout / Extended]

  Stop loss:         $[stop]  ([X]% below entry) -- [STRUCTURE/FIXED]
  Breakout vol req:  [X]K shares (50-day avg x 1.5)

----------------------------------------------
  POSITION SIZING  (based on $[account] account)
----------------------------------------------
  Risk per trade:    $[risk_dollars]  (1% of account)
  Shares:            [N] shares
  Position value:    $[value]  ([X]% of account)

----------------------------------------------
  PROFIT TARGETS
----------------------------------------------
  2R target:         $[target_2r]  (+[X]% from entry)
  3R target:         $[target_3r]  (+[X]% from entry)
  20% move:          $[target_20]

----------------------------------------------
  EXIT WATCH
----------------------------------------------
  * Hard stop: Close below $[stop] -> exit full position
  * Climax top: Large up day (3x avg move) on massive volume -> trim 50%
  * 50 MA break: Close below 50 MA after extended run -> reassess
  * Time stop: No progress in 15 days -> consider exiting

[Assumptions: account size = $X (assumed); data as of [date]]
=============================================
```

---

## Step 5 — Save the Trade Plan to File

After generating the plan (for BOTH stocks and ETFs), ALWAYS save it as a .md file.

### File Output Rules

**Folder structure**:
```
Trade Plans/
  DD-MM-YYYY/
    TICKER (Company Name) - Strategy.md
```

**Examples**:
- `Trade Plans/16-04-2026/DY (Dycom Industries) - Minervini.md`
- `Trade Plans/16-04-2026/TQQQ (ProShares 3x Nasdaq) - RSI-Based.md`
- `Trade Plans/16-04-2026/SPY (S&P 500 ETF) - Mean Reversion.md`
- `Trade Plans/16-04-2026/RGTI (Rigetti Computing) - Minervini.md`

**Strategy names for filenames**:
- Stocks: `Minervini`
- ETF Mean Reversion: `Mean Reversion`
- ETF MA Crossover: `MA Crossover`
- ETF RSI-Based: `RSI-Based`
- ETF Volatility-Based: `Volatility-Based`

**File content**: The .md file should contain the FULL trade plan (same content shown in chat), formatted in clean Markdown. Use the same structure as the chat output but with proper Markdown headers and formatting.

**Procedure**:
1. Generate the plan and display it in chat FIRST
2. Create the date folder if it doesn't exist: `Trade Plans/DD-MM-YYYY/`
3. Write the .md file with the full plan content
4. Link the file to the user so they can access it

**Date format**: Use DD-MM-YYYY for the folder name (day first, matching the user's preference).

**Company name lookup**: If you don't know the company name, use the ticker description from Massive MCP or a reasonable short name. For ETFs, use the fund's common short name (e.g., "ProShares 3x Nasdaq" for TQQQ, "S&P 500 ETF" for SPY).

---

## Tone and Judgment

The goal is to hand the user a complete, honest picture — not to talk them into or out of a trade. When violations exist, explain *why* that criterion matters (e.g., "200 MA not trending up means the stock hasn't established a sustained uptrend yet — these setups have lower odds"). Keep it factual, not preachy.

If the stock is a strong pass, say so clearly. If it's borderline, call it borderline. Minervini's own rule: "When in doubt, sit it out."

When data is limited or ambiguous (e.g., the VCP structure isn't clear), say so rather than forcing a conclusion. A hedge is better than false confidence.

---

## SEPA Rule Reminders

- **No Stage 1, 3, or 4 trades.** If the Trend Template fails materially (3+ criteria), make the Stage failure prominent.
- **No chasing.** If price > 5% above pivot, mark as Extended and do not give entry guidance.
- **Volume confirmation is required.** A breakout without 50%+ volume surge above the 50-day average is suspect — note this.
- **Account size assumption.** If you don't know the user's account size, use $100,000 and note it clearly.
