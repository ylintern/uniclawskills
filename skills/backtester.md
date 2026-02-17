---
name: backtester
version: 1.0.0
role: Historical simulation and strategy validation
requires: SKILL.md
---

# Backtester — Role Skill

Validates strategies on historical data before any live capital is used.
The gatekeeper. If it doesn't pass backtest, it doesn't get deployed.

---

## Role Objective

Given a strategy spec, simulate it on historical price data and return
a clear verdict: **PASS** or **FAIL** with evidence.

---

## Backtest Workflow

```
1. RECEIVE strategy spec from Strategist or UniClaw master
2. LOAD price data for the target pool and time range
3. SIMULATE position over time:
     For each candle/block:
       - Is position in range? → accrue fees
       - Did price cross a boundary? → trigger rebalance
       - Apply gas cost on each rebalance
       - Track: fees, IL, rebalances, capital efficiency
4. COMPUTE metrics
5. COMPARE vs baseline (HODL or previous strategy)
6. RETURN verdict with full data
```

## Simulation Engine

```python
class BacktestResult:
    strategy_name: str
    period: str                  # "2024-01-01 → 2024-12-31"
    initial_capital: float

    # Performance
    final_value: float
    total_fees: float
    total_il: float
    total_gas: float
    net_profit: float            # fees - IL - gas
    roi_pct: float

    # Risk
    sharpe_ratio: float
    max_drawdown: float
    sortino_ratio: float

    # Efficiency
    time_in_range_pct: float     # capital efficiency
    n_rebalances: int
    avg_fee_per_day: float
    win_rate: float              # % of periods profitable

    # Verdict
    verdict: str                 # PASS | FAIL
    verdict_reason: str
```

## Backtest Report Format

```
BACKTEST REPORT
═══════════════
Strategy:    [name]
Pool:        [pair + fee]
Period:      [start → end] ([N] days)
Capital:     $[initial]

RETURNS
  Net Profit:    $[N] ([N]% ROI)
  vs HODL:       +[N]% (or -[N]%)
  Fees earned:   $[N]
  IL incurred:   -$[N]
  Gas spent:     -$[N]

RISK
  Sharpe Ratio:  [N]
  Max Drawdown:  [N]%
  Sortino Ratio: [N]

EFFICIENCY
  Time in range: [N]%
  Rebalances:    [N] (avg cost: $[N])
  Avg fees/day:  $[N]

VERDICT: [✅ PASS | ❌ FAIL]
Reason:  [One sentence]
```

## Pass Thresholds

```python
PASS_CRITERIA = {
    "net_profit": lambda x: x > 0,
    "roi_vs_hodl": lambda x: x > 0,          # must beat HODL
    "sharpe_ratio": lambda x: x > 1.0,
    "max_drawdown": lambda x: x < 0.20,       # 20% max
    "time_in_range": lambda x: x > 0.60,      # 60% efficiency min
}
# ALL criteria must pass for PASS verdict
```

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial role definition | — | — |
