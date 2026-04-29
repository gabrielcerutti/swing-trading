# Watchlist

Tickers that aren't ready to trade today but are worth watching for a specific trigger. This file replaces the "No Signal" plan format — when trade-plan-generator runs on a ticker with no current setup but a clear future trigger, it adds a row here instead of writing a full plan to `Trade Plans/`.

**Mirrored to eToro:** every row in this file is also added to the eToro **Swing Watch** watchlist (`watchlistId: bd8b7764-2811-4811-a0c6-1e3490223643`) via `etoro_add_watchlist_items` so the candidate is visible on eToro mobile/web. When a row is removed here (thesis broke, pruned, or trigger fired and converted to a position), also call `etoro_remove_watchlist_item`. The local file remains the source of truth for trigger conditions and notes — the eToro watchlist is a visibility layer.

Review this list during every market open and during weekly review. Remove rows once the trigger fires (the ticker either gets a real plan or the thesis breaks).

---

## Active Watch

| Ticker | Strategy | Trigger Level | Trigger Condition | Notes | Added Date |
|---|---|---|---|---|---|

*(empty)*

---

## Field definitions

- **Strategy** — which methodology applies when triggered: `Minervini`, `Mean Reversion (ETF)`, `MA Crossover`, `RSI-Based`, `Volatility-Based`, `Scarface`
- **Trigger Level** — the specific price the ticker needs to reach. Always concrete — never "around X" or "near Y". Example: `$30.10`, `RSI(2) < 10`, `VIX > 25 + green close`
- **Trigger Condition** — what has to be true at that level for the watch to convert to a trade plan. Example: `Break of pivot on volume ≥ 1.5× 50-day avg`, `Pullback to 10 MA after 3+ down days above 200 MA`
- **Notes** — base/setup quality, expected catalysts, anything that informs how to size the plan when the trigger fires
- **Added Date** — when the row was created. If a row sits here for 8+ weeks without firing, prune it during weekly review (the setup is probably stale).

---

## When to add a row

- Ticker passes the Trend Template but is currently extended → watch for a new base
- Ticker is mid-base, pivot not yet clear → watch and update Trigger Level once the pivot forms
- ETF in pullback phase, hasn't hit the buy condition yet → watch for the specific signal
- A stock you previously traded that's setting up again

## When NOT to add a row

- A failed setup you're hoping will rebuild ("hopium watch") — let it go
- A name you "like" with no specific trigger — that's a research note, not a watchlist entry
- More than ~15 active rows — the watchlist gets ignored when it's a dump pile. Keep it pruned.

---

## Migrated from `Trade Plans/16-04-2026/`

Previous "No Signal" plans converted to watchlist entries:

| Ticker | Strategy | Trigger Level | Trigger Condition | Notes | Added Date |
|---|---|---|---|---|---|
| IGV | RSI-Based | RSI(2) < 10 with price above 200 MA | Bull-regime extreme oversold pullback | iShares Tech-Software ETF. Was overbought (RSI(2) = 100) on 2026-04-16. Wait for pullback. | 2026-04-16 |
| UFO | Mean Reversion (ETF) | Pullback to $49.50–$50.50 after 3+ red days | Procure Space ETF. Above 200 MA, looking for short-term reversion entry. | 2026-04-16 |
