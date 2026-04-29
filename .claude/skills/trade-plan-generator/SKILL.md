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

**Index ETFs**: SPY, QQQ, IWM, DIA, VOO, VTI, IVV, RSP, MDY, VB, ONEQ, SPLG, ITOT, IJH, IJR, VONE
**Style/Factor**: VUG, VTV, VONG, VONV, IWF, IWD, MTUM, USMV, QUAL, SCHD, SCHG, SCHV, VYM, VIG
**Leveraged/Inverse**: TQQQ, SQQQ, SPXL, SPXS, UPRO, SPXU, TNA, TZA, QLD, SSO, UDOW, SDOW, LABU, LABD, SOXL, SOXS, FNGU, FNGD, TECL, TECS, NAIL, DRN, JNUG, JDST, GUSH, DRIP, ERX, ERY, CURE, RXL
**Sector ETFs**: XLK, XLF, XLE, XLV, XLI, XLC, XLRE, XLP, XLU, XLB, XLY, SMH, IBB, KRE, KBE, XOP, GDX, GDXJ, XBI, ARKK, ARKG, ARKW, ARKQ, ARKF, IGV, SOXX, ITB, XHB, XRT, XME, IYR, IYT
**Thematic**: UFO, ROBO, BOTZ, CIBR, HACK, ICLN, TAN, LIT, REMX, URA, JETS, AWAY
**International**: EEM, EFA, FXI, VWO, INDA, EWZ, EWJ, EWY, EWT, EWG, EWU, EWC, EWA, EZU, ASHR, MCHI, KWEB, INDY
**Commodity**: GLD, SLV, USO, UNG, DBA, DBC, PALL, PPLT, COPX, IAU
**Bond**: TLT, IEF, SHY, HYG, LQD, AGG, BND, JNK, EMB, TIP, MUB, BIL, SHV
**Currency/Vol**: UUP, UDN, FXE, FXY, FXB, VIXY, UVXY, SVXY

### Routing Rules

```
IF ticker is in the ETF list above OR ends in common ETF patterns:
   → Use the ETF Swing Trading skill (etf-swing-trading)
   → Run the auto-selection logic to pick the best of the 4 strategies
   → Let the user override if they want a different strategy

IF ticker is an individual stock:
   → Use Minervini SEPA framework (below)

IF uncertain:
   → Call mcp__yahoo-finance__get_stock_info and check the `quoteType` field: "ETF" → ETF path, "EQUITY" → stock path
   → Ask the user if still ambiguous
```

When routing to the ETF skill, follow ALL instructions in the `etf-swing-trading` SKILL.md for strategy selection, analysis, and output format.

When routing to Minervini SEPA, continue with Steps 1-5 below.

---

## Step 1 — Fetch Data (Stocks — SEPA)

Fetch from three MCP sources. Run in parallel where possible.

**Massive MCP** (`mcp__Massive_Market_Data__query_data` / `mcp__Massive_Market_Data__call_api`):
- Daily OHLCV (close, high, low, volume) — minimum 252 bars
- Current price and today's volume
- 50-day average volume (for breakout volume confirmation)

**Yahoo Finance MCP** (`mcp__yahoo-finance__get_stock_info`):
- **Next earnings date** — field `earningsTimestamp` (Unix seconds; convert to calendar date). Fall back to `earningsTimestampStart` if `earningsTimestamp` is absent. There is **no `nextEarningsDate` field** in this MCP — do not look for it. If the resolved earnings date is within 5 trading days of the planned entry, **refuse the plan** and include the earnings date in the refusal. This check is mandatory for every stock plan; ETFs skip it. (Rule: `Risk Rules.md` §6)
- **Sector and industry** — fields `sector`, `industry`. Map to the canonical sector tags from `Open Positions.md` ("Sector field" section) for the guardrail check in Step 2.5.

**eToro MCP** (`mcp__etoro__etoro_search_instruments` → `mcp__etoro__etoro_get_rates`):
- Live bid/ask for current price confirmation. Use `exactSymbol: true` in the search to resolve the ticker to an `instrumentId`, then call `get_rates`.

**If any MCP is unavailable:**
- Massive unavailable → ask the user to paste in price data; do not proceed without it.
- Yahoo Finance unavailable → refuse the plan; the earnings window cannot be verified and the rule is non-negotiable for live trading.
- eToro unavailable → use the Massive close price as the current price; note the fallback in the plan output.

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

Apply the `VCPDetector` logic from `minervini-swing-trading` SKILL.md (the canonical implementation). Look at the last 60 trading days for a base structure and report:

- **Contraction count** (minimum 2 to qualify; 3–4 ideal)
- **Contraction depths** in order — each must be shallower than the previous (allow up to 10% tolerance per `validate_contraction_depths`)
- **Volume dry-up ratio**: 5-day avg volume vs. 20-day avg volume before the base — must be lower (use `calculate_volume_dryup`)
- **Pivot price**: the high of the last contraction (`calculate_pivot`)
- **Tightness classification**: tight (<8%), moderate (8–15%), or wide-and-loose (>15%) based on the last contraction's depth

If the base fails the VCP rules (insufficient contractions, contractions widening, volume not contracting), say so plainly — don't issue an entry plan against a failed base. This is a hard gate, not a judgment call.

### Entry Zone

- **Pivot**: High of the last contraction
- **Buy zone**: Pivot to Pivot x 1.05
- **Extended**: Current price > Pivot x 1.05 -> do NOT recommend entry
- **Not yet**: Current price < Pivot -> note "awaiting breakout"

---

## Step 2.5 — Portfolio Guardrail Check (MANDATORY before issuing any plan)

### eToro Portfolio Sync

Before running the checks, query eToro for the live state:

1. Call `mcp__etoro__etoro_get_portfolio` — returns the full account: every open position (including long-term buy-and-hold, crypto, etc.) plus `credit` (equity).
2. Call `mcp__etoro__etoro_get_trade_history` with `minDate` = today's date — for the daily realized loss check.
3. **Filter to swing positions only.** The eToro account contains many non-swing positions. Apply BOTH filters from `Open Positions.md` "Filtering eToro positions" section:
   - **Whitelist:** keep only `positionId`s that appear in `Open Positions.md`.
   - **Heuristic safety net:** if a row in `Open Positions.md` is missing its `Position ID` (legacy data), additionally accept eToro positions where `stopLossRate > openRate × 0.85`.
   - Anything that fails both checks is buy-and-hold and **must not** be counted toward swing-trade guardrails.
4. **Reconcile filtered set against `Open Positions.md`.** If counts still differ, flag the discrepancy to the user ("eToro shows X swing positions after filtering; Open Positions.md shows Y — using filtered eToro data"). Always use the filtered eToro view for guardrail math.
5. **Set account equity** based on `Risk Rules.md` §9 toggle:
   - `paper` mode → equity = $100,000 (or whatever value is set in the Risk Rules toggle).
   - `live` mode → equity = `etoro_get_portfolio().credit`.
   - If `live` is set but eToro MCP is unavailable, fall back to the last known credit value cached in `Open Positions.md` header and note the fallback. State the mode in the plan output's POSITION SIZING block.

### Guardrail Checks

Read `Open Positions.md` and `Trade Journal.md` (or eToro data from the sync above) and verify the new trade won't breach any portfolio-level limit from `Risk Rules.md` §3. **If any check fails, refuse to issue the plan and explain which limit blocked it.** Do not issue a "but be careful" plan — refuse cleanly.

Run these checks in order:

1. **Cumulative open risk.** Sum `% Equity at Risk` across all rows in `Open Positions.md`. Add this trade's prospective risk (1% for Minervini, see Risk Rules.md §1 for other strategies). If total > 6%, refuse.
2. **Sector exposure.** For this trade's sector tag, sum existing position values in that sector + this trade's prospective position value. If > 30% of equity, refuse.
3. **Concurrent positions.** Count rows in `Open Positions.md`. If already 8 open, refuse.
4. **Correlation.** If a position is already open in the same sector AND same theme (e.g. NVDA open + planning AMD; or SOXL open + planning NVDA), refuse — pick one.
5. **Daily realized loss.** Sum today's `% Account` losses in `Trade Journal.md` (ground truth per CLAUDE.md precedence). Cross-check against `etoro_get_trade_history` — if they disagree by more than $1, surface the discrepancy before computing. If ≥ 2%, refuse.
6. **Weekly realized loss.** Sum this week's `% Account` losses in `Trade Journal.md` (ground truth). Same cross-check rule as daily. If ≥ 5%, refuse.

If all six checks pass, proceed to Step 3. If the user has no `Open Positions.md` data yet (paper start), state that explicitly in the plan output ("Portfolio guardrail check: no existing positions — clean slate.") so the check is visible, not silent.

---

## Step 3 — Calculate the Trade Plan

Use these inputs to build the plan. **Account equity** is set in Step 2.5 according to `Risk Rules.md` §9 toggle (`paper` → $100,000, `live` → `etoro_get_portfolio().credit`). State the mode and equity in the POSITION SIZING block of the output.

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
  PORTFOLIO CHECK
----------------------------------------------
  Open positions:        [N] / 8 max
  Cumulative open risk:  [X.X]% / 6.0% max
  Sector exposure:       [Sector]: [X.X]% / 30.0% max
  Realized loss today:   [X.X]% / 2.0% max
  Realized loss week:    [X.X]% / 5.0% max
  Earnings in window:    [Yes (date) / No]
  Status:                [PASS / BLOCKED — reason]

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
  POSITION SIZING  (based on $[account] equity — from eToro)
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

## Step 5 — Output Routing: Plan File vs. Watchlist vs. Refusal

The output destination depends on the result of the analysis:

### Case A — Active signal (entry-ready or in buy zone)

A full trade plan goes to a dated `Trade Plans/` folder AND the user is offered to record the position in `Open Positions.md` once they execute.

### Case B — No signal yet, but a clear future trigger exists

Examples: stock passes Trend Template but is in early base, no pivot yet; ETF in bull regime but RSI(2) > 30 (no extreme oversold yet); valid setup waiting for pullback to 10 MA.

**Do not write a full plan file.** Instead:
1. Append a row to `Watchlist.md` with the trigger level and condition.
2. Resolve the ticker to its eToro `instrumentId` via `etoro_search_instruments` (use `exactSymbol: true`) — read-only call, always allowed.
3. **Sync to Swing Watch** — gated by `Risk Rules.md` §9 `ETORO_API_MODE`:
   - `read_write`: call `etoro_add_watchlist_items` with `watchlistId: bd8b7764-2811-4811-a0c6-1e3490223643` and the resolved `instrumentId`.
   - `read_only` (current): print a Manual action block — `Add [TICKER] (instrumentId: [N]) to "Swing Watch" on eToro.` Continue with the chat assessment regardless; the local `Watchlist.md` row is the source of truth.
4. Display a brief assessment in chat ("On the watchlist — trigger: [level / condition]. [Added to Swing Watch / Manual: add to Swing Watch on eToro].") rather than the full plan template.

**When pruning a Watchlist.md row** (thesis broke, stale, or trigger fired and converted to a position):
- `read_write`: call `etoro_remove_watchlist_item` with the same `watchlistId` and the row's `instrumentId`.
- `read_only`: print `Manual: remove [TICKER] (instrumentId: [N]) from "Swing Watch" on eToro.`

### Case C — Setup is broken / refused

Examples: Stage 1/3/4 stock; portfolio guardrail breach; earnings within 5 days; extended >5%.

Display the refusal in chat with the specific reason. Do not write to `Trade Plans/` and do not add to `Watchlist.md`. Optionally suggest re-checking the ticker after the blocking condition clears.

---

### File Output Rules (Case A only)

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
- Stocks (SEPA): `Minervini`
- Stocks (Scarface mean-reversion): `Scarface`
- ETF Mean Reversion: `Mean Reversion`
- ETF MA Crossover: `MA Crossover`
- ETF RSI-Based: `RSI-Based`
- ETF Volatility-Based: `Volatility-Based`

**File content**: The .md file should contain the FULL trade plan (same content shown in chat), formatted in clean Markdown. Use the same structure as the chat output but with proper Markdown headers and formatting. Always include the PORTFOLIO CHECK section.

**Procedure**:
1. Generate the plan and display it in chat FIRST
2. Create the date folder if it doesn't exist: `Trade Plans/DD-MM-YYYY/`
3. Write the .md file with the full plan content
4. Link the file to the user so they can access it
5. End with: *"When you execute, tell me the fill price and I'll record it in `Open Positions.md`."*

**Date format**: Use DD-MM-YYYY for the folder name (day first, matching the user's preference).

**Company name lookup**: If you don't know the company name, use the ticker description from Massive MCP or a reasonable short name. For ETFs, use the fund's common short name (e.g., "ProShares 3x Nasdaq" for TQQQ, "S&P 500 ETF" for SPY).

---

## Step 6 — Recording Execution (when the user reports a fill)

When the user says "I entered X at Y" (or similar):

1. Append a row to `Open Positions.md` with the fill details (Ticker, **Position ID** from the eToro fill response, Strategy, Entry Date, Entry $, Stop $, Stop $ Risk, % Equity at Risk, Sector, Current $ = Entry $ at this point, Unrealized R = 0, brief Notes).
2. Re-run the portfolio guardrail check after the new row is added — confirm no limit was breached by the actual fill (the planned vs. filled price can differ).
3. **Sync watchlists** — gated by `Risk Rules.md` §9 `ETORO_API_MODE`:
   - `read_write`:
     - Add `instrumentId` to **Swing Active** (`watchlistId: 2fb85f6b-9281-4523-8ee3-5a7b1e57c0d6`) via `etoro_add_watchlist_items`.
     - If the ticker was on Swing Watch, remove via `etoro_remove_watchlist_item` and delete the row from `Watchlist.md`.
   - `read_only` (current): print a Manual action block —
     - `Add [TICKER] (instrumentId: [N]) to "Swing Active" on eToro.`
     - If applicable: `Remove [TICKER] from "Swing Watch" on eToro.`
     - Always delete the corresponding `Watchlist.md` row locally (no API needed).
4. Confirm the update to the user with the post-trade portfolio totals.

When the user says "I exited X at Y" (or similar):

1. Compute R Multiple, $ P&L, % Account from the row in `Open Positions.md`.
2. Append a row to `Trade Journal.md` with the closed-trade details. Ask the user for the Lesson if it isn't obvious.
3. Remove the row from `Open Positions.md`.
4. **Sync watchlists** — gated by `Risk Rules.md` §9 `ETORO_API_MODE`:
   - `read_write`: call `etoro_remove_watchlist_item` against Swing Active (`watchlistId: 2fb85f6b-9281-4523-8ee3-5a7b1e57c0d6`).
   - `read_only` (current): print `Manual: remove [TICKER] (instrumentId: [N]) from "Swing Active" on eToro.`
   If the user wants to keep watching for a re-entry, offer to add it back to `Watchlist.md` + Swing Watch (which goes through the same gated flow).
5. Note that `Performance Stats.md` is now stale; offer to regenerate it.

---

## Step 7 — Trade Execution (user-triggered)

Trigger: user says "execute", "enter", "place the order", "place a limit order", or similar.

**Branch on `Risk Rules.md` §9 `ETORO_API_MODE`:**

### `read_only` mode (current)

The agent does NOT place orders. Instead, print a Manual order spec block to the user:

```
═══════════════════════════════════════════════
  MANUAL ORDER REQUIRED — eToro
═══════════════════════════════════════════════
  Ticker:        [TICKER]
  Instrument ID: [N]  (resolved via etoro_search_instruments)
  Side:          BUY
  Order type:    [Limit / Market]
  Amount (USD):  $[position_value]
  Limit price:   $[pivot]   (only for limit orders)
  Stop loss:     $[stop]
  Mode:          [demo / real per ETORO_TRADING_MODE]
═══════════════════════════════════════════════
Place this order in the eToro app, then reply with the fill details
(actual fill price + positionId from the eToro confirmation).
```

After the user confirms the manual fill, run Step 6 (manual entry recording) to update `Open Positions.md` and the watchlists.

### `read_write` mode

**Before calling any execution tool**, state the current `ETORO_TRADING_MODE` to the user ("Placing order in **demo** mode" or "Placing order in **REAL** mode — this uses live money"). Stop and ask for explicit confirmation if mode is `real`.

1. Resolve the ticker to an eToro `instrumentId` via `mcp__etoro__etoro_search_instruments` (use `exactSymbol: true`). Confirm the instrument name matches.
2. For a **limit order at pivot**: call `mcp__etoro__etoro_place_limit_order` with `amount` = position value in USD (from Step 3), `rate` = pivot price, `isBuy: true`, `stopLossRate` = stop price from the plan.
3. For a **market order**: call `mcp__etoro__etoro_open_position` with `amount` = position value in USD, `isBuy: true`, `stopLossRate` = stop price from the plan.
4. On success:
   - Record the position in `Open Positions.md` using the fill details from the eToro response — be sure to capture the returned `positionId` in the `Position ID` column.
   - Add the `instrumentId` to **Swing Active** (`watchlistId: 2fb85f6b-9281-4523-8ee3-5a7b1e57c0d6`) via `etoro_add_watchlist_items`.
   - If the ticker was on Swing Watch (`watchlistId: bd8b7764-2811-4811-a0c6-1e3490223643`), remove it via `etoro_remove_watchlist_item` and delete the corresponding row from `Watchlist.md`.
   - Confirm to the user with post-trade portfolio totals.
5. On failure: report the eToro error. Do NOT update `Open Positions.md` and do NOT touch the watchlists.

---

## Step 8 — Exit Execution (user-triggered)

Trigger: user says "close", "exit", "stop hit", "take profit", or similar for an open position.

**Branch on `Risk Rules.md` §9 `ETORO_API_MODE`:**

### `read_only` mode (current)

The agent does NOT close positions. Print a Manual close spec block:

```
═══════════════════════════════════════════════
  MANUAL CLOSE REQUIRED — eToro
═══════════════════════════════════════════════
  Ticker:        [TICKER]
  Position ID:   [from Open Positions.md]
  Close type:    [Full / Partial — units: X]
  Reason:        [stop hit / target hit / climax top / 50-MA break / time stop / manual]
  Mode:          [demo / real per ETORO_TRADING_MODE]
═══════════════════════════════════════════════
Close this position in the eToro app, then reply with the actual exit
price (and remaining units, if partial).
```

After the user confirms the manual close, run Step 6 (manual exit recording) to update `Trade Journal.md`, remove from `Open Positions.md`, and sync the watchlists.

### `read_write` mode

**Before calling any execution tool**, confirm which position to close if multiple tickers are open.

1. Call `mcp__etoro__etoro_get_portfolio` to get the live `positionId` for the ticker.
2. State `ETORO_TRADING_MODE` before executing. For `real` mode, ask for confirmation.
3. Call `mcp__etoro__etoro_close_position` with the `positionId`. Omit `unitsToDeduct` for a full close; provide it for a partial close.
4. On success:
   - Use the P&L from the eToro response to populate `Trade Journal.md` (R Multiple, $ P&L, % P&L).
   - Remove the row from `Open Positions.md`.
   - Remove the `instrumentId` from **Swing Active** (`watchlistId: 2fb85f6b-9281-4523-8ee3-5a7b1e57c0d6`) via `etoro_remove_watchlist_item`.
   - Ask the user for the Exit Reason (constrained: stop hit, target hit, climax top, 50-MA break, time stop, manual, other) and Lesson.
   - For partial closes only: keep the row in `Open Positions.md` with reduced shares; do NOT remove from Swing Active until the position is fully closed.
5. Note that `Performance Stats.md` is now stale; offer to regenerate it.

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
- **Account size.** Use the `Risk Rules.md` §9 toggle. In `paper` mode, equity = $100,000. In `live` mode, equity = `etoro_get_portfolio().credit` (fall back to cached value if eToro is unavailable). Always print the mode and equity in the plan output.
