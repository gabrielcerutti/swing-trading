# Trade Journal

Closed trades. One row per completed round-trip. Trade-plan-generator reads this file in Step 2.5 to check today's and this week's realized loss against the daily/weekly caps in `Risk Rules.md`. `Performance Stats.md` is regenerated from this file.

**Account size for % calculations:** controlled by `Risk Rules.md` §9 toggle. In `paper` mode (default), use $100,000. In `live` mode, capture the `etoro_get_portfolio().credit` value at trade time and store it in the row so % calculations remain reproducible. Match the value in `Open Positions.md`.

---

## Closed Trades

| Ticker | Strategy | Entry Date | Exit Date | Entry $ | Exit $ | Stop $ | Shares | R Multiple | $ P&L | % Account | Exit Reason | Lesson |
|---|---|---|---|---|---|---|---|---|---|---|---|---|

*(no closed trades yet)*

---

## Field definitions

- **R Multiple** = `(Exit $ - Entry $) / (Entry $ - Stop $)` for longs. A trade exited at the original 2R target = +2R; stopped out = −1R; exited at half-stop loss = −0.5R.
- **$ P&L** = `(Exit $ - Entry $) × Shares` (gross, before commissions)
- **% Account** = `$ P&L / Account Size × 100`
- **Exit Reason** — choose one (constrained vocabulary):
  - `stop hit`
  - `target hit`
  - `climax top` (Minervini sell signal)
  - `50-MA break` (Minervini sell signal)
  - `time stop`
  - `manual` (discretionary exit before any rule fired)
  - `other` (use only with explanation in Lesson column)
- **Lesson** — one sentence on what this trade taught. The lesson column is the entire point of journaling — fill it honestly. Include "no lesson" only if the trade was textbook.

---

## How to use this file

1. **On every exit**: append a row immediately. Don't wait until the end of the day, don't batch — the lesson is sharpest right after the exit.
2. **For weekly review**: filter rows by exit date for the week, compute win rate and avg R, copy into `Weekly Review.md`.
3. **For stats**: regenerate `Performance Stats.md` from this entire file.

If the same lesson appears 3 times across different trades, that's a system-level issue — flag it in the next weekly review.
