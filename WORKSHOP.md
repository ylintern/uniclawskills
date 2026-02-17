# UniClaw Workshop 01 — Uniswap Quant Intelligence
### Learn → Backtest → Execute → Collect

> This is Workshop 01 of the UniClaw skill improvement program.
> Read this file from top to bottom in a single session.
> At the end of each phase you will have produced a concrete output.
> The loop repeats — every result you collect feeds the next workshop.

---

## Before You Start

You are UniClaw — a DeFi-native quant agent whose job is to reason about
Uniswap liquidity, build quantitative signals from on-chain data, and improve
your own capabilities by running structured learning loops.

**Load order:** Read this entire file before writing any code or queries.
The framework comes first; the data layer second; execution third.

**Session state:** Record your findings in `STATE.md` as you work through
each phase. Use the tag `WORKSHOP_01:` to mark new entries.

---

# PHASE 1 — LEARN

## 1A. The Quantitative Framework

### Why Factors Work

Signals that predict returns must be **persistent, pervasive, robust, and
investable** after costs. In DeFi, the same discipline applies to pool
selection and LP allocation.

The five equity factors translate to DeFi as follows:

| Classic Factor | DeFi Equivalent | Measurement |
|---------------|-----------------|-------------|
| Momentum | Pool price momentum | 30-day log return of pool price |
| Value | Fee yield | `fees_24h / tvl * 365` (fee APR) |
| Quality | Consistency | Sharpe of daily fee income, low IL variance |
| Size | Pool depth | Total liquidity in USD |
| Low Volatility | Stable pool | 30-day realised price volatility (lower = safer LP) |

### Signal Scoring (Cross-Sectional Z-Score)

For any metric, normalise across your pool universe on each date:

```python
import numpy as np

def zscore(values):
    arr = np.array(values, dtype=float)
    return (arr - arr.mean()) / (arr.std() + 1e-9)

# Example: score 50 pools on fee APR + momentum
fee_apr_z    = zscore([pool['fee_apr'] for pool in pools])
momentum_z   = zscore([pool['momentum_30d'] for pool in pools])
composite    = 0.5 * fee_apr_z + 0.5 * momentum_z
ranked_pools = sorted(zip(composite, pools), reverse=True)
```

### Key Performance Metrics

```
Sharpe Ratio   = (mean_return - risk_free) / std_return  × √252
Sortino Ratio  = (mean_return - risk_free) / downside_std × √252
Max Drawdown   = max(peak - trough) / peak  over full period
Fee APR        = fees_collected_24h / tvl × 365 × 100
IL (approx)    = 2√(P_ratio) / (1 + P_ratio) - 1   where P_ratio = P_final/P_initial
```

### Risk Rules (Always Apply)

- Never allocate more than 20% of capital to a single pool
- Exit or reduce if pool drawdown exceeds 15% from entry
- Pause allocation to any pool where whale LP concentration > 60%
- Use Kelly fraction for position sizing:
  `f = (p × b - q) / b` where `p` = win rate, `b` = avg win/loss ratio

---

## 1B. The Data Layer

### Endpoints Directory

```
# The Graph — replace <KEY> with your API key
V4_MAINNET = "https://gateway.thegraph.com/api/<KEY>/subgraphs/id/DiYPVdygkfjDWhbxGSqAQxwBKmfKnkWQojqeM2rkLb3G"
V3_MAINNET = "https://gateway.thegraph.com/api/<KEY>/subgraphs/id/5zvR82QoaXYFyDEKLZ9t6v9adgnptxYpKpSbxtgVENFV"
V2_MAINNET = "https://gateway.thegraph.com/api/<KEY>/subgraphs/id/A3Np3RQbaBA6oKJgiwDJeo5T3zrYfGHPWFYayMwtNDum"
V1_MAINNET = "https://gateway.thegraph.com/api/<KEY>/subgraphs/id/ESnjgAG9NjfmHypk4Huu4PVvz55fUwpyrRqHF21thoLJ"

# DefiLlama — no key required
DEFILLAMA_TVL     = "https://api.llama.fi/protocol/uniswap-v3"
DEFILLAMA_VOLUMES = "https://api.llama.fi/overview/dexs"
DEFILLAMA_PRICE   = "https://coins.llama.fi/prices/current/ethereum:<TOKEN_ADDR>"

# Uniswap Trading API
TRADING_QUOTE = "https://trading-api.gateway.uniswap.org/v1/quote"
```

### The One Query Template

```javascript
async function query(endpoint, gql, variables = {}) {
  const res = await fetch(endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query: gql, variables }),
  });
  const { data, errors } = await res.json();
  if (errors) throw new Error(JSON.stringify(errors));
  return data;
}
```

### Schema Self-Learning Rule

**Before every new subgraph or version — introspect first:**

```graphql
# Step 1: list all entity types
{ __schema { queryType { fields { name } } } }

# Step 2: inspect the entity you need
{ __type(name: "Pool") { fields { name type { name } } } }
```

Never assume field names transfer between v2, v3, and v4.
Always check. Always record surprises in `STATE.md` as `WORKSHOP_01: SCHEMA_NOTE`.

### Core Queries

**Top pools by TVL (v3):**
```graphql
{
  pools(first: 50, orderBy: totalValueLockedUSD, orderDirection: desc) {
    id
    feeTier
    token0 { symbol }
    token1 { symbol }
    totalValueLockedUSD
    volumeUSD
    feesUSD
    txCount
    sqrtPrice
    tick
    liquidity
  }
}
```

**90-day daily OHLCV for a pool (v3):**
```graphql
query PoolHistory($poolId: ID!, $startTime: Int!) {
  poolDayDatas(
    where: { pool: $poolId, date_gte: $startTime }
    orderBy: date, orderDirection: asc, first: 365
  ) {
    date
    open
    high
    low
    close
    volumeUSD
    feesUSD
    tvlUSD
    txCount
  }
}
```

**Tick-level liquidity distribution (v3):**
```graphql
query Ticks($poolId: String!) {
  ticks(where: { pool: $poolId, liquidityNet_not: "0" },
    orderBy: tickIdx, orderDirection: asc, first: 1000) {
    tickIdx
    liquidityNet
    liquidityGross
    price0
    price1
  }
}
```

**LP positions in a pool (v3):**
```graphql
query Positions($poolId: String!) {
  positions(where: { pool: $poolId, liquidity_gt: "0" },
    orderBy: liquidity, orderDirection: desc, first: 100) {
    id
    owner
    tickLower { tickIdx }
    tickUpper { tickIdx }
    liquidity
    depositedToken0
    depositedToken1
    collectedFeesToken0
    collectedFeesToken1
  }
}
```

**v4 — always check hooks:**
```graphql
{
  pools(first: 20, orderBy: totalValueLockedUSD, orderDirection: desc) {
    id
    hooks
    feeTier
    currency0 { symbol }
    currency1 { symbol }
    totalValueLockedUSD
    volumeUSD
  }
}
```
> `hooks != "0x000...0"` means non-standard pool behaviour. Never aggregate
> hooked and non-hooked pools of the same pair without separate analysis.

### Pagination — Always Paginate

```javascript
// Keyset pagination (faster than skip for large datasets)
let lastId = "";
while (true) {
  const { swaps } = await query(endpoint, `
    query($lastId: ID!) {
      swaps(first: 1000, where: { id_gt: $lastId }, orderBy: id) {
        id timestamp amountUSD
      }
    }`, { lastId });
  if (swaps.length === 0) break;
  process(swaps);
  lastId = swaps.at(-1).id;
}
```

### Price Conversion

```python
def sqrt_price_x96_to_price(sqrt_price_x96, decimals0, decimals1):
    price = (int(sqrt_price_x96) / (2**96)) ** 2
    return price * (10**decimals0) / (10**decimals1)

def tick_to_price(tick, decimals0, decimals1):
    return (1.0001 ** tick) * (10**decimals0) / (10**decimals1)
```

---

## 1C. Signal Library (Five On-Chain Signals)

These are the five signals you will compute in Phase 2:

```python
import numpy as np

# S1 — Fee APR (value / yield signal)
def fee_apr(fees_24h_usd, tvl_usd):
    return (fees_24h_usd / tvl_usd) * 365 * 100 if tvl_usd > 0 else 0

# S2 — Volume / TVL ratio (activity / momentum signal)
def vol_tvl_ratio(vol_usd, tvl_usd):
    return vol_usd / tvl_usd if tvl_usd > 0 else 0

# S3 — Price momentum (trend signal)
def momentum(prices, lookback=30):
    if len(prices) < lookback: return 0
    return (prices[-1] / prices[-lookback]) - 1

# S4 — TWAP deviation (mean-reversion signal)
def twap_deviation(prices, window=24):
    if len(prices) < window: return 0
    twap = np.mean(prices[-window:])
    return (prices[-1] - twap) / twap if twap > 0 else 0

# S5 — Whale LP concentration (risk signal — lower = safer)
def whale_concentration(positions, n=5):
    liq = [int(p['liquidity']) for p in positions]
    total = sum(liq)
    return sum(sorted(liq, reverse=True)[:n]) / total if total > 0 else 0
```

---

# PHASE 2 — BACKTEST

## Objective

Test whether the five signals above actually predict which pools produce
better fee income over a 30-day forward window.

## Data Requirements

- Minimum 90 days of `poolDayData` for each pool in your universe
- Minimum 20 pools (50 is better for cross-sectional z-scoring)
- Use the v3 mainnet subgraph

## Walk-Forward Backtest Protocol

**Critical rule: never use random k-fold. Finance requires temporal splits.**

```
Full history: [==========================================]
                                                         
Window 1:  Train [-------60 days-------] | Test [30d]
Window 2:        Train [-------60 days-------] | Test [30d]
Window 3:              Train [-------60 days-------] | Test [30d]
...

Gap between train end and test start = 0 days minimum for daily data
(For intraday data, gap = 5× signal half-life to avoid leakage)
```

## Backtest Script

```python
import numpy as np
from datetime import datetime, timedelta

def run_backtest(pool_history_dict, start_date, end_date,
                 train_days=60, test_days=30):
    """
    pool_history_dict: { pool_id: [ {date, close, feesUSD, tvlUSD, volumeUSD} ] }
    Returns: list of (window_start, ic_fee_apr, ic_vol_tvl, ic_momentum)
    """
    results = []
    current = start_date

    while current + timedelta(days=train_days + test_days) <= end_date:
        train_end = current + timedelta(days=train_days)
        test_end  = train_end + timedelta(days=test_days)

        scores, fwd_returns = [], []

        for pool_id, history in pool_history_dict.items():
            train = [d for d in history if current <= datetime.fromtimestamp(d['date']) < train_end]
            test  = [d for d in history if train_end <= datetime.fromtimestamp(d['date']) < test_end]
            if len(train) < 20 or len(test) < 5: continue

            prices = [d['close'] for d in train]
            fee_apr_val   = fee_apr(train[-1]['feesUSD'], train[-1]['tvlUSD'])
            vol_tvl_val   = vol_tvl_ratio(train[-1]['volumeUSD'], train[-1]['tvlUSD'])
            momentum_val  = momentum(prices, 30)

            # Forward return = total fees collected in test window / starting TVL
            fwd_fee_return = sum(d['feesUSD'] for d in test) / train[-1]['tvlUSD']

            scores.append((fee_apr_val, vol_tvl_val, momentum_val))
            fwd_returns.append(fwd_fee_return)

        if len(scores) < 10:
            current += timedelta(days=test_days)
            continue

        # Rank IC (Spearman rank correlation of signal vs forward return)
        from scipy.stats import spearmanr
        fa  = [s[0] for s in scores]
        vtl = [s[1] for s in scores]
        mom = [s[2] for s in scores]

        ic_fa,  _ = spearmanr(fa,  fwd_returns)
        ic_vtl, _ = spearmanr(vtl, fwd_returns)
        ic_mom, _ = spearmanr(mom, fwd_returns)

        results.append({
            'window_start': current.isoformat(),
            'n_pools': len(scores),
            'ic_fee_apr': round(ic_fa, 4),
            'ic_vol_tvl': round(ic_vtl, 4),
            'ic_momentum': round(ic_mom, 4),
        })

        current += timedelta(days=test_days)

    return results
```

## What Good Looks Like

| Metric | Weak | Acceptable | Strong |
|--------|------|------------|--------|
| Mean IC | < 0.03 | 0.03–0.08 | > 0.08 |
| IC > 0 rate | < 55% | 55–65% | > 65% |
| IC Sharpe (IC / std_IC) | < 0.3 | 0.3–0.5 | > 0.5 |

> **IC below 0.03 across all windows:** The signal has no predictive power.
> Discard it and try a different metric. Do not force a bad signal into production.

## Backtest Output Format

```json
{
  "workshop": "01",
  "phase": "backtest",
  "date_run": "2026-02-17",
  "universe": "v3_top50_by_tvl",
  "period": "2025-06-01 to 2026-02-01",
  "windows_tested": 8,
  "signals": {
    "fee_apr":    { "mean_ic": 0.091, "ic_positive_rate": 0.75, "ic_sharpe": 0.61 },
    "vol_tvl":    { "mean_ic": 0.054, "ic_positive_rate": 0.63, "ic_sharpe": 0.38 },
    "momentum_30d": { "mean_ic": 0.047, "ic_positive_rate": 0.60, "ic_sharpe": 0.31 }
  },
  "composite_weights": { "fee_apr": 0.50, "vol_tvl": 0.30, "momentum_30d": 0.20 },
  "notes": ""
}
```

Save this file as `evals/backtest_results_workshop01.json`.

---

# PHASE 3 — EXECUTE

## Objective

Apply the composite signal to produce a live pool ranking and LP allocation
recommendation. This is not a trade order — it is a structured recommendation
for human review before capital is deployed.

## Execution Pipeline

```
[1] Fetch current top-50 v3 pools
        │
[2] For each pool → fetch last 30 days of PoolDayData
        │
[3] Compute: fee_apr, vol_tvl_ratio, momentum_30d
        │
[4] Z-score each signal cross-sectionally
        │
[5] Composite score = weighted sum (use weights from backtest output)
        │
[6] Rank pools → select top quintile (top 10 of 50)
        │
[7] Risk filter:
    - Remove pools where whale_concentration > 0.60
    - Remove pools where fee_apr = 0 (pool inactive)
    - Remove v4 pools with non-zero hooks unless hook type is known safe
        │
[8] Allocate capital using risk parity:
    - Each pool's weight ∝ 1 / volatility_30d
    - Normalise weights to sum to 1
    - Apply 20% single-pool cap
        │
[9] Output recommendation JSON (see format below)
```

## Risk Parity Allocation

```python
def risk_parity_weights(vols, max_single=0.20):
    """Allocate inversely proportional to volatility."""
    inv_vol = np.array([1/v if v > 0 else 0 for v in vols])
    weights = inv_vol / inv_vol.sum()
    # Apply cap and renormalise
    weights = np.minimum(weights, max_single)
    return weights / weights.sum()
```

## Execution Output Format

```json
{
  "workshop": "01",
  "phase": "execute",
  "timestamp": "2026-02-17T00:00:00Z",
  "signal_weights": { "fee_apr": 0.50, "vol_tvl": 0.30, "momentum_30d": 0.20 },
  "universe_size": 50,
  "selected_pools": [
    {
      "rank": 1,
      "pool_id": "0x...",
      "pair": "WETH/USDC",
      "fee_tier": "0.05%",
      "composite_score_z": 1.82,
      "fee_apr": 18.4,
      "vol_tvl_ratio": 0.31,
      "momentum_30d": 0.12,
      "tvl_usd": 98000000,
      "volatility_30d": 0.024,
      "whale_concentration": 0.38,
      "risk_parity_weight": 0.18,
      "risk_flags": [],
      "recommendation": "LP"
    }
  ],
  "excluded_pools": [
    { "pool_id": "0x...", "reason": "whale_concentration > 0.60" }
  ],
  "portfolio_metrics": {
    "expected_fee_apr_weighted": 14.2,
    "max_single_pool_weight": 0.18,
    "pools_selected": 8
  }
}
```

Save as `evals/execution_workshop01_<YYYYMMDD>.json`.

---

# PHASE 4 — COLLECT RESULTS

## Objective

30 days after execution, measure what actually happened. Compare realised
outcomes against predicted scores. Update signal weights for the next workshop.

## What to Measure

For each pool in your execution output, collect:

```python
def collect_results(pool_id, entry_date, exit_date, entry_tvl, subgraph_endpoint):
    """Fetch actual realised metrics for the 30-day window."""
    history = fetch_pool_day_data(pool_id, entry_date, exit_date, subgraph_endpoint)

    actual_fee_income  = sum(d['feesUSD'] for d in history)
    actual_fee_apr     = actual_fee_income / entry_tvl * 365 * 100
    actual_vol_usd     = sum(d['volumeUSD'] for d in history)
    price_start        = history[0]['open']
    price_end          = history[-1]['close']
    price_return       = (price_end / price_start) - 1

    # Impermanent loss approximation (for 50/50 pool)
    price_ratio = price_end / price_start
    il = (2 * np.sqrt(price_ratio) / (1 + price_ratio)) - 1

    return {
        "pool_id": pool_id,
        "actual_fee_apr": round(actual_fee_apr, 2),
        "actual_vol_usd": round(actual_vol_usd, 0),
        "price_return_30d": round(price_return * 100, 2),
        "impermanent_loss_approx": round(il * 100, 2),
        "net_lp_return": round((actual_fee_income / entry_tvl - abs(il)) * 100, 2),
    }
```

## Results Output Format

```json
{
  "workshop": "01",
  "phase": "collect",
  "collection_date": "2026-03-19",
  "window": "2026-02-17 to 2026-03-19",
  "results": [
    {
      "pool_id": "0x...",
      "pair": "WETH/USDC",
      "predicted_composite_z": 1.82,
      "actual_fee_apr": 16.1,
      "actual_price_return": -3.2,
      "impermanent_loss": -1.1,
      "net_lp_return": 2.4
    }
  ],
  "portfolio_summary": {
    "avg_fee_apr_realised": 13.8,
    "avg_il": -0.9,
    "avg_net_lp_return": 2.1,
    "rank_ic_fee_apr": 0.72,
    "rank_ic_composite": 0.81
  },
  "signal_update": {
    "fee_apr":    { "old_weight": 0.50, "new_weight": 0.52, "reason": "highest IC realised" },
    "vol_tvl":    { "old_weight": 0.30, "new_weight": 0.28, "reason": "IC slightly lower" },
    "momentum_30d": { "old_weight": 0.20, "new_weight": 0.20, "reason": "unchanged" }
  }
}
```

Save as `evals/results_workshop01_<YYYYMMDD>.json`.

---

## Signal Weight Update Rule

After collecting results, update composite weights using IC-weighted averaging:

```python
def update_weights(old_weights, realised_ics, smoothing=0.3):
    """
    Blend old weights with new IC-proportional weights.
    smoothing: how much to move toward new weights (0=no change, 1=full replace)
    """
    total_ic = sum(max(ic, 0) for ic in realised_ics.values())
    new_weights = {
        signal: max(ic, 0) / total_ic if total_ic > 0 else 1/len(realised_ics)
        for signal, ic in realised_ics.items()
    }
    blended = {
        signal: (1 - smoothing) * old_weights[signal] + smoothing * new_weights[signal]
        for signal in old_weights
    }
    total = sum(blended.values())
    return {k: round(v / total, 3) for k, v in blended.items()}
```

---

# CLOSING THE LOOP

## What to Write in STATE.md After This Workshop

```markdown
## WORKSHOP_01 — Completed [DATE]

### Signals Validated
- fee_apr IC: [value] — [keep/discard/reduce weight]
- vol_tvl IC: [value] — [keep/discard/reduce weight]
- momentum_30d IC: [value] — [keep/discard/reduce weight]

### Updated Composite Weights
fee_apr: [X] | vol_tvl: [X] | momentum_30d: [X]

### Schema Notes
- [Any v3 field name surprises]
- [Any v4 hooks encountered]

### Next Workshop Candidates
- [ ] Add whale_concentration as a risk filter score
- [ ] Test TWAP deviation as a mean-reversion signal
- [ ] Expand universe to include v4 non-hooked pools
- [ ] Add DefiLlama TVL trend as a macro filter
```

## Workshop Progression

Each workshop builds on the last:

```
Workshop 01 (this file)
  Learn: Quant framework + Uniswap data layer
  Backtest: 5 signals on v3 top-50 pools
  Execute: Risk-parity LP allocation
  Collect: IC, realised APR, IL, net return
        │
        ▼
Workshop 02
  Add: TWAP deviation signal
  Add: DefiLlama TVL trend as macro filter
  Extend: v4 pool universe (non-hooked only)
  Improve: Walk-forward with adaptive weights
        │
        ▼
Workshop 03
  Add: On-chain LP flow signal (mint/burn ratio)
  Add: Cross-pool arb spread signal (v2 vs v3 price)
  Build: Automated result collector (scheduled query)
```

---

## Files to Commit After This Workshop

```
skills/quant-trading-knowledge.md         ← quantitative framework KB
skills/uniswap-queries-knowledge.md       ← on-chain data layer KB
evals/backtest_results_workshop01.json    ← your backtest output
evals/execution_workshop01_YYYYMMDD.json  ← your execution output
evals/results_workshop01_YYYYMMDD.json    ← your collected results
STATE.md                                  ← updated with WORKSHOP_01 notes
```

---

*UniClaw Workshop 01 | https://github.com/ylintern/uniclawskills | MIT License*
