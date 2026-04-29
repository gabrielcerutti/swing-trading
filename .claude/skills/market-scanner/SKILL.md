---
name: market-scanner
description: Scans the S&P 500 and/or Nasdaq 100 for Stage 2 stocks passing the Minervini Trend Template, then ranks candidates by VCP tightness and proximity to pivot. Use this skill whenever the user wants to screen the market, find setups, scan for breakouts, build a watchlist, or asks things like "what's setting up right now?", "find me some VCP candidates", "run a scan", "what stocks are in Stage 2?", or "give me a shortlist to trade". Also trigger when the user mentions scanning, screening, or filtering any stock universe. Always use this skill before asking the user to name specific tickers — the scan finds the candidates.
---

# Market Scanner

Scan a stock universe for Minervini Stage 2 / VCP candidates using parallel sub-agents, then rank and present the best setups. After showing results, ask the user whether to generate full trade plans on the top picks.

---

## Overview of the Workflow

1. **Resolve the universe** — get the ticker list (S&P 500, Nasdaq 100, or both)
2. **Split into batches** — divide tickers across 4–6 parallel sub-agents
3. **Each sub-agent screens its batch** — runs Trend Template + basic VCP check
4. **Collect and rank results** — merge passing tickers, sort by quality
5. **Present the shortlist** — top candidates in chat
6. **Save to CSV** — full results to the Swing Trading folder
7. **Offer trade plans** — ask if the user wants plans on the top picks

---

## Data Requirements

| Field | Source | Tool |
|---|---|---|
| Daily OHLCV (252+ bars) | Massive MCP | `mcp__Massive_Market_Data__query_data` |
| Current price | Massive MCP | most recent close from OHLCV |
| Universe / constituents | Massive MCP | `mcp__Massive_Market_Data__call_api` |
| Sector and industry | Yahoo Finance | `mcp__yahoo-finance__get_stock_info` → `sector`, `industry` (fetched per passing ticker after Trend Template screen) |

If Massive MCP is unavailable, the scanner cannot run — say so and do not produce a partial result. If Yahoo Finance is unavailable, proceed with sector = "Unknown" and note it in the output.

---

## Step 1 — Resolve the Universe

Default universe: **S&P 500 + Nasdaq 100** (~550 unique tickers after deduplication).

If the user specifies a different universe (e.g., "just Nasdaq", "my watchlist: AAPL, MSFT, NVDA"), use that instead.

To get ticker lists, use Massive MCP via `mcp__Massive_Market_Data__call_api`. Common endpoints:

- S&P 500 constituents
- Nasdaq 100 constituents

If Massive MCP doesn't have a constituents endpoint, use these hardcoded fallback lists (abbreviated — use the real full lists in practice):

**S&P 500 core names to always include**: AAPL, MSFT, NVDA, AMZN, META, GOOGL, BRK.B, JPM, V, UNH, XOM, JNJ, PG, MA, HD, CVX, MRK, ABBV, LLY, PEP, KO, AVGO, COST, TMO, MCD, ACN, DHR, NEE, TXN, QCOM, RTX, HON, LIN, IBM, SBUX, GS, AMD, INTC, CAT, GE, SPGI, AXP, ISRG, VRTX, REGN, GILD, SLB, PLD, AMT

**Nasdaq 100 additions**: TSLA, NFLX, ADBE, PYPL, MELI, PANW, CRWD, SNOW, DDOG, ZS, OKTA, TTD, TEAM, FTNT, WDAY, VEEV, DOCU, ZM, SPLK, MDB, NET, BILL, GTLB, HUBS, CFLT

Deduplicate the combined list before scanning.

---

## Step 2 — Split into Batches and Spawn Sub-Agents

Divide the ticker list into **4–6 equal batches**. For ~550 tickers, use 5 batches of ~110 tickers each.

Spawn all batch agents **in the same message** (parallel). Each agent receives:
- Its batch of tickers
- Instructions to run the Trend Template screen (Step 3 below)
- Instructions to return results as a JSON list

Each sub-agent prompt should be self-contained — include all the screening criteria directly in the prompt so it doesn't need to read the skill file.

### Sub-Agent Prompt Template

```
You are a market screener. For each ticker in the list below, fetch 252 days of daily OHLCV data via the Massive MCP tools, then check whether it passes the Minervini Trend Template.

Tickers: [BATCH_LIST]

For each ticker, calculate:
- MA50 = 50-day simple moving average of close
- MA150 = 150-day simple moving average of close  
- MA200 = 200-day simple moving average of close
- MA200_22d = 200-day MA value 22 trading days ago
- high_52w = highest close in last 252 days
- low_52w = lowest close in last 252 days
- current_price = most recent close

Trend Template criteria (all 8 must pass):
1. current_price > MA150
2. current_price > MA200
3. MA150 > MA200
4. MA200 > MA200_22d  (200 MA trending up)
5. MA50 > MA150 AND MA50 > MA200
6. current_price > MA50
7. current_price >= low_52w * 1.25  (25% above 52w low)
8. current_price >= high_52w * 0.75  (within 25% of 52w high)

Also calculate for each passing ticker:
- pct_from_52w_high = (current_price - high_52w) / high_52w * 100  (negative = below high)
- criteria_passed = count of passing criteria

For each ticker that PASSES all 8 criteria, also call mcp__yahoo-finance__get_stock_info to get the `sector` and `industry` fields. Use the `sector` value in the output. If Yahoo Finance is unavailable or the call fails for a ticker, set sector = "Unknown".

If data fetch fails for a ticker, do NOT skip silently. Return an entry for that ticker with the error so the main scanner can report it.

Return ONLY a JSON object with two keys, `passes` and `errors`:
{
  "passes": [
    {
      "ticker": "AAPL",
      "price": 195.50,
      "ma50": 188.20,
      "ma150": 182.10,
      "ma200": 175.40,
      "high_52w": 198.00,
      "low_52w": 142.00,
      "pct_from_52w_high": -1.3,
      "criteria_passed": 8,
      "sector": "Technology"
    }
  ],
  "errors": [
    {"ticker": "XYZ", "reason": "rate-limited after 1 retry"},
    {"ticker": "OLDCO", "reason": "fewer than 252 bars of history"}
  ]
}

Return `{"passes": [], "errors": [...]}` if nothing passes. Return only the JSON, no other text.
```

---

## Step 3 — Collect, Merge, Rank, and Audit Failures

Once all sub-agents return, merge their `passes` arrays AND their `errors` arrays.

**Audit the errors first.** Compute `failure_rate = errors_count / total_tickers_scanned`.

- If `failure_rate > 5%`: stop and report to the user. Show the failed tickers and reasons. Ask whether to retry the failed batch or proceed with a partial scan. Do not silently produce a result based on >5% missing data.
- If `failure_rate ≤ 5%`: proceed to ranking, but include the failed-ticker count in the output so the user knows the scan wasn't complete.

Then rank the `passes`:

**Primary sort**: `pct_from_52w_high` descending (closest to 52w high = tightest setup, most actionable)

**Secondary sort**: `criteria_passed` descending (more criteria = stronger trend)

**Filter out**: Any ticker where `pct_from_52w_high < -25%` (too far from highs, unlikely to have a clean VCP)

Take the **top 20** after ranking.

---

## Step 4 — Present the Shortlist in Chat

Use this exact format:

```
═══════════════════════════════════════════════
  MARKET SCAN — [UNIVERSE]             [DATE]
  [N] scanned → [M] Stage 2 candidates  ([F] failed)
═══════════════════════════════════════════════

RANK  TICKER   PRICE    FROM HIGH   CRITERIA   SECTOR
─────────────────────────────────────────────────────
  1   NVDA    $189.31    -2.1%       8/8  ⭐   Semis
  2   AAPL    $195.50    -4.5%       8/8  ⭐   Tech
  3   MSFT    $412.00    -6.2%       8/8        Tech
  4   META    $580.20    -8.1%       8/8        Internet
  5   AVGO    $215.40    -9.4%       7/8        Semis
  ...

⭐ = within 5% of 52-week high (prime buy zone)

[If F > 0:]
─────────────────────────────────────────────────────
  FAILED FETCHES ([F] tickers, [P]% of universe)
─────────────────────────────────────────────────────
  XYZ — rate-limited after 1 retry
  OLDCO — fewer than 252 bars of history
  ...

────────────────────────────────────────────
Saved full results to: Screener Results/scan_[DATE].csv
────────────────────────────────────────────

Would you like me to generate full trade plans for the top 3 setups?
```

The ⭐ flag applies to any ticker within 5% of its 52-week high — these are the most actionable candidates, closest to potential pivot breakouts.

---

## Step 5 — Save to CSV

Save the full ranked list (all Stage 2 candidates, not just top 20) to:
`Screener Results/scan_[YYYY-MM-DD].csv` (relative to the project root — `d:\Claude\Swing Trading\Screener Results\`)

CSV columns: `rank, ticker, price, ma50, ma150, ma200, high_52w, low_52w, pct_from_52w_high, criteria_passed, sector`

If failures occurred, also write `Screener Results/scan_[YYYY-MM-DD]_errors.csv` with columns `ticker, reason` so the user can audit which names were missed.

---

## Step 6 — Offer Trade Plans

After presenting the shortlist, always end with:

> "Would you like me to generate full trade plans for the top 3 setups?"

If the user says yes, run the **trade-plan-generator** skill on each of the top 3 tickers — spawn them in parallel so all 3 plans come back at once.

---

## Handling Failures and Edge Cases

**Rate limiting**: If Massive MCP returns 429 errors, have each sub-agent retry once with a short delay, then skip tickers that still fail. Note in the output how many tickers were skipped.

**Insufficient data**: Skip tickers with fewer than 252 bars of history (e.g., recent IPOs).

**Empty results**: If fewer than 5 tickers pass the Trend Template, report this honestly — it likely means the broad market is in a downtrend. Say so explicitly: "Only [N] stocks passed the Trend Template. This is a bearish signal for the overall market — consider reducing exposure."

**User-specified universe**: If the user provides their own list of tickers, use those instead of the default universe. No need to batch — just run them sequentially or as a single sub-agent if the list is small (<30 tickers).

---

## Performance Notes

- ~550 tickers across 5 parallel agents = ~110 tickers each
- Each agent fetches 252 days × 110 tickers via Massive MCP
- Expected total runtime: 3–6 minutes depending on API response times
- Let the user know upfront: "Running the scan now — this takes 3–5 minutes."
