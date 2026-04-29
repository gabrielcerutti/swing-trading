# Performance Stats

Regenerated from `Trade Journal.md`. **Do not hand-edit the stats below** — recompute the entire file from the journal whenever asked or during weekly review.

Last regenerated: *(not yet — no closed trades)*

---

## Overall

| Metric | Value |
|---|---|
| Total trades closed | 0 |
| Win rate | — |
| Avg winner (R) | — |
| Avg loser (R) | — |
| Expectancy (R per trade) | — |
| Profit factor | — |
| Largest winner ($, R) | — |
| Largest loser ($, R) | — |
| Total realized $ P&L | $0 |
| Total realized % return | 0.0% |
| Max drawdown ($, %) | — |
| Avg holding period (days) | — |

## By Strategy

| Strategy | Trades | Win % | Avg R | Total R |
|---|---|---|---|---|
| Minervini SEPA | 0 | — | — | 0.0R |
| ETF — Mean Reversion | 0 | — | — | 0.0R |
| ETF — MA Crossover | 0 | — | — | 0.0R |
| ETF — RSI-Based | 0 | — | — | 0.0R |
| ETF — Volatility-Based | 0 | — | — | 0.0R |
| Scarface Mean-Reversion | 0 | — | — | 0.0R |

## By Exit Reason

| Reason | Count | Total R |
|---|---|---|
| stop hit | 0 | 0.0R |
| target hit | 0 | 0.0R |
| climax top | 0 | 0.0R |
| 50-MA break | 0 | 0.0R |
| time stop | 0 | 0.0R |
| manual | 0 | 0.0R |
| other | 0 | 0.0R |

## Benchmark comparison

| Period | Account return | SPY return | QQQ return | Excess vs SPY | Excess vs QQQ |
|---|---|---|---|---|---|
| All-time | — | — | — | — | — |
| Year-to-date | — | — | — | — | — |
| Last 4 weeks | — | — | — | — | — |

---

## How to regenerate

1. Read every row in `Trade Journal.md`.
2. Compute each metric:
   - **Win rate** = wins / total trades
   - **Avg winner R / Avg loser R** = mean of R-multiples for winners / losers separately
   - **Expectancy** = `(win_rate × avg_winner_R) + ((1 − win_rate) × avg_loser_R)` (avg_loser_R is negative)
   - **Profit factor** = `sum(winner $ P&L) / abs(sum(loser $ P&L))`
   - **Max drawdown** = peak-to-trough drop in cumulative $ P&L (track running cumulative P&L through the journal in chronological order)
   - **Avg holding period** = mean of `Exit Date − Entry Date` in calendar days
3. For benchmarks: fetch SPY and QQQ daily closes via Massive MCP for the matching periods and compute total return.
4. Overwrite this file with the new numbers. Update the "Last regenerated" timestamp.

If a strategy has 0 closed trades, leave the row in place with `0` and `—` so it's visible the strategy hasn't been used.
