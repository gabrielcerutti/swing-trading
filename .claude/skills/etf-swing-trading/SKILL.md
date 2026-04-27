---
name: etf-swing-trading
description: Unified ETF swing trading system with 4 strategies — Mean Reversion, Moving Average Crossover, RSI-based, and Volatility-based. Use this skill whenever the user asks about trading index ETFs (QQQ, SPY, IWM, DIA, TQQQ, SQQQ, SPXL, TNA, etc.) or any leveraged/inverse ETF. Also trigger when the user mentions ETF trading, index trading, or asks for a plan on any ETF ticker. This skill auto-selects the best strategy based on current market conditions but allows user override. Do NOT use the Minervini SEPA system for ETFs — use this skill instead.
tags: etf, swing-trading, mean-reversion, moving-average, rsi, volatility, index, leveraged-etf, qqq, spy
---

# ETF Swing Trading System

Trade index and leveraged ETFs using the right strategy for current conditions. This skill contains 4 complementary strategies and auto-recommends the best one based on trend, volatility, and momentum readings.

## Why Not Minervini for ETFs?

The SEPA system (Trend Template, VCP, pivot breakouts) was designed for **individual stocks** where supply/demand dynamics, institutional accumulation, and earnings catalysts drive price. ETFs are mechanically derived from underlying indices — they don't have VCPs, earnings surprises, or shareholder rotation patterns. Leveraged ETFs add volatility decay on top of that. These 4 strategies are purpose-built for how index ETFs actually behave.

---

## Strategy Selection Logic

When the user asks for a trade plan on an ETF, run this decision tree to auto-recommend a strategy. Always show the recommendation and let the user override.

### Auto-Selection Rules

Fetch the data first (see Data Requirements below), then evaluate:

```
1. Calculate: RSI(2), RSI(14), 50 MA, 200 MA, ATR(14), VIX level (if available)

2. IF price < 200 MA AND 200 MA trending down:
   → REGIME: BEAR MARKET
   → Recommend: "No trade — index is in a downtrend. Wait for 200 MA reclaim."
   → If user insists: Volatility-Based (buy VIX spikes for counter-trend bounces only)

3. IF price > 200 MA (bull regime), select:

   a. RSI(2) < 10 AND price > 200 MA:
      → Recommend: RSI-BASED (extreme short-term oversold in uptrend)
      → This is the highest-probability setup

   b. VIX > 25 (or ATR(14) > 2× its 50-day average):
      → Recommend: VOLATILITY-BASED (fear spike = buying opportunity)

   c. Price recently crossed above 50 MA (within 5 days) AND 50 MA > 200 MA:
      → Recommend: MA CROSSOVER (fresh trend confirmation)

   d. RSI(2) > 90 AND price > 200 MA:
      → Recommend: MEAN REVERSION — SHORT-TERM EXIT / no new entry
      → "Overbought short-term. Wait for pullback before entering."

   e. None of the above triggered:
      → Recommend: MEAN REVERSION (default — wait for next pullback to buy)

4. Present recommendation + brief rationale, then ask:
   "This is my recommended strategy. Want me to proceed, or would you prefer a different one?"
```

---

## Data Requirements

For ALL strategies, fetch via Massive MCP:

- **252+ trading days** of daily OHLCV data for the ETF
- **Current price and volume**
- **50-day and 200-day moving averages**
- **RSI(2) and RSI(14)** — calculate from close prices
- **ATR(14)** — Average True Range for volatility measurement
- **VIX level** (if available via Massive MCP; if not, use ATR as volatility proxy)

### RSI Calculation

```
RSI(n) where n = period length:
1. Calculate daily price changes: change = close - prev_close
2. Separate gains (positive changes) and losses (absolute negative changes)
3. Average gain = SMA of gains over n periods
4. Average loss = SMA of losses over n periods
5. RS = average_gain / average_loss
6. RSI = 100 - (100 / (1 + RS))
```

### ATR Calculation

```
True Range = MAX(high - low, ABS(high - prev_close), ABS(low - prev_close))
ATR(14) = 14-period SMA of True Range
```

---

## Strategy 1: Mean Reversion

### Philosophy
Index ETFs have a strong statistical tendency to revert to their mean over 2-7 day periods. Buy short-term pullbacks within an uptrend, sell the bounce. Based on research by Larry Connors and Cesar Alvarez.

### Entry Rules
- **Trend filter**: Price must be ABOVE the 200-day MA (confirms bull market)
- **Pullback signal**: Price closes lower for 3+ consecutive days
- **Confirmation**: Price pulls back to or below the 10-day MA
- **Entry**: Buy at next day's open after all conditions are met

### Exit Rules
- **Profit exit**: Close position when price closes ABOVE the 5-day MA
- **Time stop**: Exit after 7 trading days if target not hit
- **Hard stop**: 5% below entry (ETFs rarely need this in uptrend pullbacks)

### Position Sizing
- Risk per trade: 1.5% of account (slightly higher than stock trades — index pullbacks are more predictable)
- Position size: `risk_dollars / (entry - stop)`
- Cap at 30% of account for standard ETFs (QQQ, SPY)
- Cap at 15% of account for leveraged ETFs (TQQQ, SPXL) — the leverage IS the position sizing

### Key Metrics to Report
- Current consecutive down days
- Distance from 10-day MA
- Distance from 200-day MA (trend health)
- Historical win rate context: "Mean reversion on SPY/QQQ after 3+ down days above the 200 MA has historically won ~70-75% of the time"

---

## Strategy 2: Moving Average Crossover

### Philosophy
Ride the intermediate trend using moving average alignment. This is a slower, position-trading approach — fewer trades, longer holds, bigger moves. Works well for core index exposure.

### Entry Rules
- **Primary signal**: 10-day MA crosses ABOVE the 30-day MA (weekly timeframe equivalent: 2-week over 6-week)
- **Trend filter**: 50-day MA must be above 200-day MA (or within 2% and rising)
- **Price confirmation**: Price must close above BOTH the 10 MA and 30 MA on the crossover day
- **Entry**: Buy at next day's open after crossover confirms

### Exit Rules
- **Exit signal**: 10-day MA crosses BELOW the 30-day MA
- **Trailing stop**: 2× ATR(14) below the highest close since entry
- **Hard stop**: 8% below entry

### Position Sizing
- Risk per trade: 2% of account (wider stops, lower frequency)
- Position size: `risk_dollars / (entry - stop)`
- Cap at 40% of account for standard ETFs (this is a trend-following approach, larger positions are appropriate)
- Cap at 20% of account for leveraged ETFs

### Key Metrics to Report
- Current MA alignment (10 vs 30 vs 50 vs 200)
- Days since last crossover
- ATR-based trailing stop level
- Whether this is a fresh cross or an established trend

---

## Strategy 3: RSI-Based (Connors RSI)

### Philosophy
The RSI(2) strategy identifies extreme short-term oversold conditions in index ETFs. When RSI(2) drops below 10 while the long-term trend is intact, the bounce probability is very high. This is the most data-backed ETF swing strategy.

### Entry Rules
- **Trend filter**: Price ABOVE 200-day MA (non-negotiable)
- **Signal**: RSI(2) closes below 10
- **Aggressive entry**: Buy at next day's open when RSI(2) < 10
- **Conservative entry**: Wait for RSI(2) < 5 for even higher probability

### Exit Rules
- **Primary exit**: RSI(2) closes above 65 → sell next day's open
- **Stretch target**: RSI(2) closes above 80 → definitely sell
- **Time stop**: Exit after 5 trading days if RSI hasn't hit 65
- **Hard stop**: 4% below entry (tight — these are short-duration trades)

### Position Sizing
- Risk per trade: 1% of account
- Position size: `risk_dollars / (entry - stop)`
- Cap at 25% for standard ETFs
- Cap at 12% for leveraged ETFs
- Note: These are SHORT holding period trades (2-5 days typically)

### Key Metrics to Report
- Current RSI(2) value
- Current RSI(14) value (for context)
- How many days RSI(2) has been below 10
- Price distance from 200 MA
- Historical context: "RSI(2) < 10 with price above 200 MA on QQQ: avg bounce of 2-4% within 5 days"

---

## Strategy 4: Volatility-Based

### Philosophy
Buy fear, sell calm. When volatility spikes (VIX surges, ATR expands dramatically), index ETFs tend to be near short-term bottoms. This strategy times entries around volatility extremes.

### Entry Rules
- **Trend filter**: Price above 200-day MA OR within 5% of it (allows entries during sharp corrections)
- **Volatility signal (primary)**: VIX closes above 25 AND has risen 30%+ from its 20-day average
- **Volatility signal (backup)**: If VIX unavailable, use ATR(14) > 1.5× its 50-day average
- **Confirmation**: Wait for first green close (close > open) after volatility spike
- **Entry**: Buy at next day's open after green-candle confirmation

### Exit Rules
- **Primary exit**: VIX drops below 20 (or ATR returns to within 1.1× its 50-day average)
- **Profit target**: 5-8% gain from entry
- **Time stop**: Exit after 10 trading days
- **Hard stop**: 6% below entry

### Position Sizing
- Risk per trade: 1% of account (volatility entries are inherently riskier)
- Position size: `risk_dollars / (entry - stop)`
- Cap at 20% for standard ETFs
- Cap at 10% for leveraged ETFs — these are counter-trend entries in volatile markets
- **Scale in**: Consider entering 50% at signal, 50% if it dips further within 2 days

### Key Metrics to Report
- Current VIX level (or ATR vs average)
- VIX percentile (where current VIX sits vs last 252 days)
- Whether the green candle confirmation has occurred
- Distance from 200 MA

---

## Output Format

Use this layout for ALL ETF trade plans. Adapt the strategy-specific sections based on which strategy is selected.

```
═══════════════════════════════════════════════
  ETF TRADE PLAN — [TICKER]            [DATE]
═══════════════════════════════════════════════

📊 STRATEGY: [Mean Reversion / MA Crossover / RSI-Based / Volatility-Based]
   Auto-selected because: [brief rationale]

[If applicable:]
⚠️  CONCERNS
──────────────
• [Any warnings — e.g., leveraged ETF decay risk, trend weakness, etc.]

──────────────────────────────────────────────
  MARKET REGIME
──────────────────────────────────────────────
  Price vs 200 MA:   $[price] vs $[ma200]  [Bull/Bear/Neutral]
  Price vs 50 MA:    $[price] vs $[ma50]
  50 MA vs 200 MA:   [Above/Below/Converging]
  RSI(2):            [value]  [Oversold <10 / Neutral / Overbought >90]
  RSI(14):           [value]
  ATR(14):           $[value]  ([X]× its 50-day avg)
  VIX:               [value or "N/A"]
  Regime:            [BULL TREND / PULLBACK IN UPTREND / BEAR / VOLATILE]

──────────────────────────────────────────────
  [STRATEGY NAME] SIGNAL
──────────────────────────────────────────────
  [Strategy-specific criteria — show each rule and pass/fail]
  Signal status:     [ACTIVE / PENDING / NO SIGNAL]

──────────────────────────────────────────────
  ENTRY & RISK
──────────────────────────────────────────────
  Current price:     $[price]
  Entry trigger:     [specific condition that triggers the buy]
  Entry price:       $[entry or "at open when triggered"]

  Stop loss:         $[stop]  ([X]% below entry)
  Expected hold:     [X-Y] trading days

──────────────────────────────────────────────
  POSITION SIZING  (based on $[account] account)
──────────────────────────────────────────────
  Risk per trade:    $[risk_dollars]  ([X]% of account)
  Shares:            [N] shares
  Position value:    $[value]  ([X]% of account)
  [If leveraged ETF:]
  Effective exposure: $[value × leverage]  ([X]% equivalent)

──────────────────────────────────────────────
  PROFIT TARGETS & EXIT
──────────────────────────────────────────────
  [Strategy-specific exit rules with concrete price levels]
  Time stop:         [X] trading days

──────────────────────────────────────────────
  STRATEGY NOTES
──────────────────────────────────────────────
  [Historical context, win rate info, what to watch for]

  [If leveraged ETF, always include:]
  ⚠️ LEVERAGE WARNING: [TICKER] is a [X]x daily leveraged ETF.
  Volatility decay erodes value in choppy markets. Do not hold
  for extended periods. A [stop%]% stop = only a [stop%/leverage]%
  move in the underlying index.

[Assumptions: account size = $X; data as of [date]]
═══════════════════════════════════════════════
```

---

## Leveraged ETF Adjustments

When the ticker is a leveraged ETF (TQQQ, SQQQ, SPXL, SPXS, TNA, TZA, UPRO, etc.):

1. **Halve all position size caps** (the leverage IS the sizing)
2. **Tighten stops by 30%** (a 7% stop becomes ~5%)
3. **Shorten time stops by 40%** (hold period 5 days becomes 3 days)
4. **Always include the leverage warning** in the output
5. **Calculate effective exposure**: position value × leverage factor
6. **Note volatility decay**: "Holding TQQQ for more than 5-10 days in a volatile market introduces meaningful decay. These are short-duration trades."

### Common Leveraged ETFs Reference
| Ticker | Underlying | Leverage |
|--------|-----------|----------|
| TQQQ | Nasdaq 100 | 3x Long |
| SQQQ | Nasdaq 100 | 3x Short |
| SPXL | S&P 500 | 3x Long |
| SPXS | S&P 500 | 3x Short |
| UPRO | S&P 500 | 3x Long |
| TNA | Russell 2000 | 3x Long |
| TZA | Russell 2000 | 3x Short |
| QLD | Nasdaq 100 | 2x Long |
| SSO | S&P 500 | 2x Long |
| UDOW | Dow 30 | 3x Long |

---

## Tone

Same as the Minervini trade plan generator: direct, honest, no sugar-coating. If the market regime says "don't trade," say so. If a leveraged ETF is inappropriate for the setup, say so. The user wants clarity, not encouragement.

If no strategy produces an active signal, say: "No signal right now. Here's what each strategy is waiting for: [brief list]." Don't force a trade.
