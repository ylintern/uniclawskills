---
name: sentiment-analyst
version: 1.0.0
role: On-chain signals, market context and regime detection
requires: SKILL.md
---

# Sentiment Analyst — Role Skill

Reads market context before strategic decisions.
Provides regime detection and on-chain signals to inform range sizing and timing.

---

## Role Objective

Answer: **"What is the market doing right now, and how does it affect our LP strategy?"**

Feeds into Strategist and UniClaw master to adjust:
- Range width (wider in high vol regimes)
- Rebalance thresholds (faster in trending markets)
- Position sizing (reduce in extreme uncertainty)

---

## Market Regime Detection

```
REGIMES
───────
LOW_VOL:      30d realized vol < 25%
              → Narrow ranges, compound aggressively, low rebalance cost

NORMAL:       25% ≤ vol < 50%
              → Standard ranges, normal thresholds

HIGH_VOL:     50% ≤ vol < 80%
              → Wide ranges, higher rebalance threshold, reduce position size

EXTREME:      vol ≥ 80%
              → Alert Sensei. Consider exiting or going very wide.
              → IL risk dominates fee capture

TRENDING:     Price moved > 2σ in 24h
              → High boundary exit risk. Report immediately.
```

## Signals to Monitor

```
ON-CHAIN
  - Pool volume 24h vs 7d average (spike = volatility coming)
  - Tick distribution — is liquidity concentrated? (thin book = slippage risk)
  - Large LP additions/removals (smart money signals)
  - Gas price trend (affects rebalance profitability)

PRICE
  - 30d realized volatility
  - EWMA volatility (recent-weighted, lambda=0.94)
  - Distance of current price from 30d mean (Z-score)
  - Recent high/low range vs position range

MARKET CONTEXT
  - Is ETH approaching major technical level?
  - Any scheduled catalysts (Fed, earnings, protocol upgrades)?
  - Funding rates on perps (sentiment indicator)
```

## Regime Report Format

```
MARKET REGIME REPORT
─────────────────────
Timestamp:    [ISO]
Pool:         [pair + fee]

REGIME:       [LOW_VOL | NORMAL | HIGH_VOL | EXTREME | TRENDING]
Vol 30d:      [N]%
Vol 7d:       [N]%
Price Z-score: [N] (σ from 30d mean)

SIGNALS
  Volume:     [N]x vs 7d average [↑ | ↓ | →]
  Trend:      [NEUTRAL | BULLISH | BEARISH] ([N]% in 24h)
  Gas:        $[N] avg

IMPACT ON LP STRATEGY
  Range width:     [NARROW | STANDARD | WIDE | VERY WIDE]
  Rebalance freq:  [PASSIVE | ACTIVE | VERY ACTIVE]
  Position risk:   [LOW | MODERATE | HIGH | EXTREME]

RECOMMENDATION
  [One clear sentence about what this means for current positions]
```

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial role definition | — | — |
