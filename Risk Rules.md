# Risk Rules

**Single source of truth for all risk, sizing, and portfolio limits.** Every skill in this project references this file. If a number in a SKILL.md disagrees with what's here, this file wins.

Last reviewed: 2026-04-27.

---

## 1. Per-Trade Risk by Strategy

| Strategy | Risk per trade | Hard stop max | Stop preference |
|---|---|---|---|
| Minervini SEPA (stocks) | 1% of equity | 8% below entry | Structure: 1% below last contraction low |
| ETF — Mean Reversion | 1.5% of equity | 5% below entry | Fixed % (mean reversion bounces are quick) |
| ETF — MA Crossover | 2% of equity | 8% below entry | 2x ATR(14) trailing once in profit |
| ETF — RSI-Based | 1% of equity | 4% below entry | Fixed % (short-duration trades) |
| ETF — Volatility-Based | 1% of equity | 6% below entry | Fixed % |
| Scarface Mean-Reversion (stocks) | 1% of equity total across scaled entries | 3σ + 1 ATR beyond mean | Statistical |

**Rule for Minervini specifically:** if structure-based stop is wider than 8%, use 8% — but consider whether the setup is worth taking with that much room. Tighter is better.

---

## 2. Position Size Caps by Strategy

| Strategy | Standard ETF / liquid stock cap | Leveraged ETF cap |
|---|---|---|
| Minervini SEPA | 25% of account | N/A (no leveraged stocks) |
| ETF — Mean Reversion | 30% | 15% |
| ETF — MA Crossover | 40% | 20% |
| ETF — RSI-Based | 25% | 12% |
| ETF — Volatility-Based | 20% | 10% |
| Scarface Mean-Reversion | 25% across all scaled entries | N/A |

**The cap is a ceiling, not a target.** Position size is `risk_dollars / (entry - stop)`, then bounded by the cap. The risk budget usually determines size before the cap kicks in.

---

## 3. Portfolio-Level Limits (Apply Across All Strategies)

These limits span the entire account. Trade-plan-generator must check these before issuing any plan (see Step 2.5 in trade-plan-generator SKILL.md).

| Limit | Threshold | Source for the check |
|---|---|---|
| Max cumulative open risk | 6% of equity | Sum of `% Equity at Risk` column in `Open Positions.md` |
| Max sector exposure | 30% of equity | Sum of position values per sector |
| Max concurrent positions | 8 | Row count in `Open Positions.md` |
| Max correlated names | 1 per sector + theme | E.g. don't open both NVDA and AMD if SOXL is already open |
| Max realized loss today | 2% of equity | Sum of today's closed P&L in `Trade Journal.md` |
| Max realized loss this week | 5% of equity | Sum of this week's closed P&L in `Trade Journal.md` |

**If any limit would be breached, refuse the plan and explain which limit blocked it.** Don't issue a "you could trade this but be careful" plan — refuse cleanly.

---

## 4. The "Extended" Rule (Stocks)

> Current price > pivot × 1.05 → **Extended. Do not enter.**

This applies everywhere: CLAUDE.md, minervini-swing-trading SKILL.md, trade-plan-generator SKILL.md. The 5% cap is non-negotiable. Wait for a new base.

(For ETFs, "extended" isn't a defined concept — strategy-specific signals govern entries.)

---

## 5. Volume Confirmation

| Setup | Volume rule |
|---|---|
| Minervini breakout | Day-of volume ≥ 50-day avg × 1.5 (50%+ above average) |
| ETF Mean Reversion entry | No specific volume rule (pullbacks happen on light volume by design) |
| ETF MA Crossover | No specific volume rule |
| ETF RSI-Based | No specific volume rule |
| ETF Volatility-Based | Green-candle confirmation after VIX spike (price action, not volume) |
| Scarface Mean-Reversion | Climactic volume = exhaustion signal — note in plan but don't add as entry filter |

For Minervini specifically: **a breakout without 50%+ volume is suspect.** The plan must flag this.

---

## 6. Earnings Window (Stocks Only)

> No new entries within 5 trading days before earnings. Holding through earnings is allowed only if the position is up >2R and a portion has already been taken off.

ETFs don't have earnings — this rule doesn't apply.

For Minervini, Scarface, and any stock plan: trade-plan-generator must query the next earnings date during data fetch (Step 1) and refuse the plan if entry would land inside the 5-day window. Document the earnings date in any refusal.

---

## 7. Leveraged ETF Adjustments

When the ticker is leveraged (TQQQ, SQQQ, SPXL, SPXS, TNA, TZA, UPRO, QLD, SSO, UDOW, SDOW, LABU, LABD, SOXL, SOXS, FNGU, FNGD, TECL, TECS):

1. Halve the position size cap (already reflected in the table above)
2. Tighten the stop by 30% (a 7% stop becomes ~5%)
3. Shorten time stops by 40% (a 5-day hold becomes 3 days)
4. Always include a leverage decay warning in the output
5. Compute and report effective exposure: `position_value × leverage_factor`

---

## 8. Mean Reversion: Which Approach Applies?

The project has two mean-reversion methodologies. Use the right one for the instrument:

- **ETF Mean Reversion strategy** (inside `etf-swing-trading`): pullback-based. 3+ consecutive down days while above the 200 MA. Use this for **index and sector ETFs only.**
- **Scarface mean-reversion skill** (`mean-reversion`): statistical Z-score-based. `Z ≤ -2σ AND RSI < 30`. Use this for **individual liquid stocks already in Stage 2** — never for stocks in Stage 1, 3, or 4 (the no-bottom-fishing rule still applies).

Don't apply Scarface to ETFs (use the ETF version), and don't apply the ETF pullback approach to individual stocks (use Scarface for the statistical approach, or wait for a Minervini setup).

---

## 9. Account Equity Source — `paper` vs `live` Toggle

```
ACCOUNT_MODE   = paper       # change to `live` to use eToro equity
PAPER_EQUITY   = 100000      # USD; only used when ACCOUNT_MODE = paper
ETORO_API_MODE = read_only   # `read_only` (current) or `read_write`
```

### `ETORO_API_MODE` — read-only vs read-write

The eToro API key is **read-only** until the user explicitly flips this to `read_write`. While `read_only`:

- ✅ **Allowed:** all read tools — `etoro_get_portfolio`, `etoro_get_trade_history`, `etoro_search_instruments`, `etoro_get_rates`, `etoro_get_watchlists`, `etoro_get_user_*`, etc.
- ❌ **Blocked:** all write tools — `etoro_open_position`, `etoro_close_position`, `etoro_place_limit_order`, `etoro_cancel_order`, `etoro_add_watchlist_items`, `etoro_remove_watchlist_item`, `etoro_create_watchlist`, `etoro_delete_watchlist`, `etoro_rename_watchlist`, `etoro_set_default_watchlist`, `etoro_create_post`, `etoro_create_comment`.

**When a write would have fired, do NOT call the tool.** Instead, print a clear "Manual action required" block to the user with all the parameters they need (instrument, watchlistId, order side, amount, stop, etc.) so they can do it on the eToro app. After printing, ask the user to confirm when the manual step is done — then proceed with the local file updates (`Open Positions.md`, `Watchlist.md`, `Trade Journal.md`). Never update local files based on a write the system "would have" done — wait for confirmation that the eToro side actually happened.

When the user flips this to `read_write`, the same code paths become live API calls and the manual blocks disappear.

**Why a toggle:** the eToro account in `mcp__etoro__etoro_get_portfolio` holds long-term buy-and-hold positions that are **not** part of the swing-trading system. Using `credit` directly mixes paper-trading discipline with a real long-term portfolio. Pick one mode and trade it cleanly.

### `paper` mode (default)
- Account equity = `PAPER_EQUITY` ($100,000 unless changed above).
- Trade plans are still issued, executed via eToro **demo** mode (`ETORO_TRADING_MODE=demo`), and logged in `Trade Journal.md` against the paper equity.
- The eToro MCP is still queried for the live portfolio, but only to apply the swing-trade filter from `Open Positions.md` "Filtering eToro positions" — never as the equity base.

### `live` mode
- Account equity = `etoro_get_portfolio().credit`.
- `ETORO_TRADING_MODE` should be `real`. State this loudly before every order.
- Open Positions and Trade Journal % calculations must use the live `credit` value at trade time. Cache the value at fill time so the row's % math is reproducible later.
- If eToro MCP is unavailable in `live` mode, fall back to the last known `credit` cached in the `Open Positions.md` header. Note the fallback in the plan output.

### Hard rule
The plan output's POSITION SIZING block must always state which mode and equity were used (e.g., `POSITION SIZING (based on $100,000 paper equity — paper mode)`). No silent assumptions.

---

## 10. What This File Does NOT Cover

- Specific entry/exit signals per strategy → see each strategy's SKILL.md
- VCP detection logic → see `minervini-swing-trading` SKILL.md (`VCPDetector` class is canonical)
- Backtesting parameters → see `mean-reversion` SKILL.md `MeanReversionBacktest`

---

## Change log

- 2026-04-27: Initial consolidation. Reconciled per-trade risk %, stop max %, and position caps across all skills. Added portfolio-level limits and earnings rule.
