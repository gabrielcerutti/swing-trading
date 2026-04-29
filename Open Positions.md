# Open Positions

Active **swing trades**. Trade-plan-generator reads this file in Step 2.5 (portfolio guardrail check) before issuing any new plan. Update this file the moment a plan is executed in the broker; remove the row when the trade is closed and append the closed row to `Trade Journal.md`.

**Source of truth for swing trades:** this file (the `Position ID` column ties each row to a specific eToro `positionId`). The eToro account holds many non-swing positions (long-term buy-and-hold, crypto, etc.) — those are **not** counted toward swing-trade guardrails. See "Filtering eToro positions" below.

**Account equity source:** controlled by the toggle in `Risk Rules.md` §9. Default is paper ($100,000). When the toggle is `live`, equity = `etoro_get_portfolio().credit`. Always state which mode is in effect in the plan output.

---

## Active

| Ticker | Position ID | Strategy | Entry Date | Entry $ | Stop $ | Stop $ Risk | % Equity at Risk | Sector | Current $ | Unrealized R | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|

*(no swing positions open)*

---

## How rows are computed

- **Stop $ Risk** = `(Entry $ - Stop $) × Shares`
- **% Equity at Risk** = `Stop $ Risk / Account Size × 100`
- **Unrealized R** = `(Current $ - Entry $) / (Entry $ - Stop $)` — i.e. how many R the position is up or down
- **Position ID** = the `positionId` returned by eToro on order fill. **Required** — this is the join key for guardrail filtering.

## Filtering eToro positions (CRITICAL)

The eToro account contains long-term buy-and-hold positions that are NOT swing trades and must be excluded from guardrail math. Use both filters:

1. **Whitelist (authoritative):** A position is a swing trade if its `positionId` appears in the Active table above. Plans add a row here only on system-issued fills. Manually opened positions are never swing trades unless explicitly added.
2. **Heuristic safety net:** If the whitelist is incomplete (e.g., a manual eToro entry that should have been recorded), an eToro position should be treated as a swing trade only if `stopLossRate > openRate × 0.85` (i.e., a real stop within 15% of entry). Long-term buy-and-hold positions in this account use throwaway stops at $0.0001-$0.01 — they fail this test and are excluded.

Step 2.5 of trade-plan-generator must apply both filters when computing concurrent positions, cumulative open risk, and sector exposure. **Never count the buy-and-hold portfolio toward the 8-position cap or the 6% open-risk cap.**

## Sector field

Use these sector tags consistently so the 30% sector exposure cap can be enforced:

- `Tech`, `Semis`, `Software`, `Internet`, `Financials`, `Banks`, `Healthcare`, `Biotech`, `Energy`, `Materials`, `Industrials`, `Consumer Disc`, `Consumer Stap`, `Utilities`, `Comms`, `Real Estate`, `Index ETF`, `Sector ETF`, `Leveraged ETF`, `International`, `Commodity`, `Bond`

### Yahoo Finance sector → canonical tag mapping

`get_stock_info` returns `sector` and `industry` strings. Map them as follows when adding a row:

| Yahoo `sector` | Yahoo `industry` (if it disambiguates) | Canonical tag |
|---|---|---|
| Technology | Semiconductors, Semiconductor Equipment & Materials | `Semis` |
| Technology | Software—Application, Software—Infrastructure | `Software` |
| Technology | Internet Content & Information | `Internet` |
| Technology | (anything else, e.g. Hardware, Computers) | `Tech` |
| Communication Services | Internet Content & Information | `Internet` |
| Communication Services | (Telecom, Entertainment, Advertising) | `Comms` |
| Financial Services | Banks—Diversified, Banks—Regional | `Banks` |
| Financial Services | (anything else) | `Financials` |
| Healthcare | Biotechnology, Drug Manufacturers—Specialty & Generic | `Biotech` |
| Healthcare | (anything else) | `Healthcare` |
| Energy | (anything) | `Energy` |
| Basic Materials | (anything) | `Materials` |
| Industrials | (anything) | `Industrials` |
| Consumer Cyclical | (anything) | `Consumer Disc` |
| Consumer Defensive | (anything) | `Consumer Stap` |
| Utilities | (anything) | `Utilities` |
| Real Estate | (anything) | `Real Estate` |

For ETFs, Yahoo's sector field is usually empty — use ticker-based logic instead:
- Index ETFs (SPY, QQQ, IWM, DIA, VOO, etc.) → `Index ETF`
- Leveraged/inverse ETFs (TQQQ, SQQQ, SPXL, etc.) → `Leveraged ETF`
- Sector/thematic ETFs → `Sector ETF`
- Bond ETFs → `Bond`
- Commodity ETFs (GLD, SLV, USO, etc.) → `Commodity`
- International ETFs (EEM, EFA, FXI, etc.) → `International`

## Pre-trade checks (run before adding a row)

1. Sum the `% Equity at Risk` column → must stay ≤ 6% with the new trade added
2. Sum position values per sector → must stay ≤ 30% per sector
3. Row count ≤ 8 with new trade added
4. No correlated name already open (same Sector + same theme)
5. Today's realized loss in `Trade Journal.md` < 2% of equity

If any check fails, refuse the new plan. Cite the limit and the offending number.
