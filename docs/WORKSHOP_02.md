# UniClaw Workshop 02 — LP Operations Mastery
### Learn → Backtest → Execute → Collect

> **Prerequisites:** Workshop 01 complete. Composite signal weights saved in
> `evals/backtest_results_workshop01.json`. `STATE.md` updated.
>
> **What this workshop trains:** The full operational lifecycle of a Uniswap v3/v4
> LP position — from opening a deposit to closing and measuring true profit.
> You will learn to track inventory, calculate IL, ONL, fee income, and net P&L
> with precision. Every number must be derivable from on-chain data alone.
>
> **Tag all STATE.md entries this session:** `WORKSHOP_02:`

---

## Before You Start — The LP Operator's Mindset

An LP is not a passive holder. You are running a **market-making business** with
four ongoing responsibilities:

```
DEPOSIT      → Choose pool, range, size. Open position.
MONITOR      → Track inventory drift, fee accrual, IL, ONL daily.
MAINTAIN     → Collect fees. Rebalance range. Swap back to target inventory.
CLOSE        → Withdraw liquidity. Measure true net profit.
```

Every decision in this cycle has a cost and a return. The goal of this workshop
is to measure both precisely so you can decide whether to hold, rebalance, or exit.

---

# PHASE 1 — LEARN

## 1A. Core Concepts

### Liquidity and Price Ranges

In Uniswap v3/v4, you do not deposit into the entire price curve. You choose a
**tick range** `[tickLower, tickUpper]` and your capital is only active — earning
fees — when the pool price is inside your range.

```
Price:    |----[==============|====current====|==============]-----|
               tickLower                                    tickUpper

Inside range:  Your liquidity is active. You earn fees on every swap.
Outside range: Your position is 100% in one token. Zero fees earned. IL frozen.
```

The narrower your range, the **higher your fee APR** when in-range, but the
**more often you go out of range** and stop earning. Wider ranges earn less per
dollar but stay active longer. This is the central LP tradeoff.

### Position State at Any Moment

At any time your position can be fully described by:

```python
@dataclass
class PositionState:
    pool_id:        str
    tick_lower:     int
    tick_upper:     int
    liquidity:      int          # L — the invariant quantity
    token0_held:    float        # current token0 in position
    token1_held:    float        # current token1 in position
    fees0_owed:     float        # uncollected fees in token0
    fees1_owed:     float        # uncollected fees in token1
    entry_price:    float        # price of token1 in token0 at deposit
    current_price:  float        # price now
    entry_value_usd: float       # USD value of tokens at deposit
    current_value_usd: float     # USD value of tokens now (ex-fees)
```

### The Four P&L Components

Every LP position has exactly four P&L components. **Never conflate them.**

```
GROSS P&L = Fee Income + Price Return + IL + ONL

Where:
  Fee Income   = fees collected + fees owed (accrued but uncollected)
  Price Return = change in USD value of the token pair itself (market move)
  IL           = loss vs holding the same tokens outside the pool
  ONL          = opportunity notional loss vs the best alternative use of capital
```

You must calculate all four separately. Showing only "net P&L" hides which
component is driving performance and makes it impossible to improve.

---

## 1B. Impermanent Loss (IL) — Precise Definition

IL is the difference between the current value of your LP position and the
value you would have had if you had simply **held the same tokens** in a wallet.

```
IL = Value(LP position now) − Value(HODL same tokens from entry)
```

### IL Formula for a v3 Concentrated Position

For a full-range (v2-style) position with equal value of token0 and token1:

```python
def il_full_range(price_ratio):
    """
    price_ratio = current_price / entry_price
    Returns IL as a fraction (negative = loss vs HODL)
    """
    return (2 * (price_ratio ** 0.5) / (1 + price_ratio)) - 1
```

For a **concentrated position** in tick range [pLow, pHigh], IL is path-dependent
and depends on whether price has left the range. Use the precise formula:

```python
import math

def concentrated_il(entry_price, current_price, p_low, p_high):
    """
    All prices in same units (e.g. USDC per ETH).
    Returns IL as a fraction vs HODL.
    """
    pa, pb = p_low, p_high
    p0, p1 = entry_price, current_price

    def lp_value(price):
        """Value of LP position (normalised to L=1) at given price."""
        p = max(pa, min(price, pb))       # clamp to range
        return 2 * math.sqrt(p * pb) - p - pb if price <= pa else \
               2 * math.sqrt(pa * p) - pa - p if price >= pb else \
               2 * math.sqrt(p) * (math.sqrt(pb) - math.sqrt(pa)) - (pb - pa) / math.sqrt(p)
               # simplified — use SDK for production

    def hodl_value(price, token0_share, token1_share):
        """Value if you just held the initial split."""
        return token0_share * price + token1_share

    # At entry, LP sets token split based on range
    sqrt_p0 = math.sqrt(p0)
    sqrt_pa = math.sqrt(pa)
    sqrt_pb = math.sqrt(pb)
    token1_share = sqrt_p0 - sqrt_pa          # token0 (e.g. ETH) per unit L
    token0_share = (1/sqrt_p0 - 1/sqrt_pb)   # token1 (e.g. USDC) per unit L
    # Note: token0/token1 naming follows Uniswap convention (lower address = token0)

    lp_now   = lp_value(p1)
    hodl_now = hodl_value(p1, token0_share, token1_share)

    return (lp_now - hodl_now) / hodl_now

# Rule of thumb: for 2x price move, full-range IL ≈ -5.7%
# For 10% range (±5%), a 2x price move causes IL ≈ -100% (full out-of-range)
```

### IL by Price Move — Reference Table

| Price Change | Full Range IL | ±10% Range IL | ±50% Range IL |
|-------------|--------------|---------------|---------------|
| +5% | -0.06% | out-of-range | -0.3% |
| +10% | -0.23% | out-of-range | -1.2% |
| +25% | -1.47% | out-of-range | -5.8% |
| +50% | -5.72% | out-of-range | -13.4% |
| +100% | -20.0% | out-of-range | out-of-range |
| -50% | -5.72% | out-of-range | -13.4% |

> **Key insight:** A narrow range earns more fees per dollar when in-range,
> but one large move puts you fully out-of-range with frozen IL and zero income.
> The fee income must exceed the IL for LP to beat HODL.

---

## 1C. Opportunity Notional Loss (ONL)

ONL answers a different question than IL:

> **IL asks:** "Did I do worse than HODL?"
> **ONL asks:** "Did I do worse than the next best thing I could have done?"

ONL = Value(LP position) − Value(best alternative strategy over same period)

The "best alternative" is context-dependent. Common benchmarks:

```python
class ONLBenchmark:
    HODL_50_50    = "hold 50% token0, 50% token1 (same value at entry)"
    HODL_100_ETH  = "hold 100% ETH — relevant if you're bullish"
    HODL_100_USDC = "hold 100% stable — relevant if you expect drawdown"
    LENDING_APY   = "deposit both tokens in Aave/Compound — risk-free DeFi yield"
    COMPETITOR_LP = "LP in equivalent pool on Curve or competing DEX"

def onl(lp_current_value, lp_fees_collected, benchmark_value):
    """
    lp_current_value: current USD value of position tokens (ex fees)
    lp_fees_collected: USD value of all fees earned
    benchmark_value: what the capital would be worth in benchmark strategy
    Returns ONL as USD (negative = you underperformed the benchmark)
    """
    lp_total = lp_current_value + lp_fees_collected
    return lp_total - benchmark_value
```

**Why ONL matters more than IL in practice:**
IL only tells you vs HODL. If ETH dropped 60%, your IL might be "only" -20%,
but both you AND the HODLer lost money. ONL vs a stablecoin yield benchmark
would show the true cost of being in the pool during a crash.

---

## 1D. Fee Collection Mechanics

### How Fees Accrue

Every swap in a pool generates fees. These are **not automatically sent** to
your wallet — they accumulate inside the pool contract as:
`feeGrowthInside0X128` and `feeGrowthInside1X128` per unit of liquidity.

Your owed fees at any moment:

```
fees0_owed = liquidity × (feeGrowthInside0_now − feeGrowthInside0_at_deposit) / 2^128
fees1_owed = liquidity × (feeGrowthInside1_now − feeGrowthInside1_at_deposit) / 2^128
```

You must call `collect()` on the position manager to realise these as tokens
in your wallet. Fees left uncollected still count toward your P&L calculation —
they are real owed value — but they are not usable capital until collected.

### Query: Uncollected Fees from Subgraph

```graphql
query PositionFees($positionId: ID!) {
  position(id: $positionId) {
    id
    liquidity
    collectedFeesToken0
    collectedFeesToken1
    feeGrowthInside0LastX128
    feeGrowthInside1LastX128
    pool {
      feeGrowthGlobal0X128
      feeGrowthGlobal1X128
      tick
      ticks(where: { tickIdx_in: [$tickLower, $tickUpper] }) {
        tickIdx
        feeGrowthOutside0X128
        feeGrowthOutside1X128
      }
    }
  }
}
```

### Compute Owed Fees Off-Chain

```python
def compute_fees_owed(liquidity, fg_global, fg_outside_lower,
                      fg_outside_upper, fg_inside_last, current_tick,
                      tick_lower, tick_upper):
    """
    All feeGrowth values are raw X128 integers from the contract.
    Returns owed fees in token units (divide by token decimals for human value).
    """
    UINT256_MAX = 2**256

    # Compute feeGrowthBelow (at tickLower)
    if current_tick >= tick_lower:
        fg_below = fg_outside_lower
    else:
        fg_below = (fg_global - fg_outside_lower) % UINT256_MAX

    # Compute feeGrowthAbove (at tickUpper)
    if current_tick < tick_upper:
        fg_above = fg_outside_upper
    else:
        fg_above = (fg_global - fg_outside_upper) % UINT256_MAX

    # feeGrowthInside
    fg_inside = (fg_global - fg_below - fg_above) % UINT256_MAX

    # Owed fees
    owed = liquidity * ((fg_inside - fg_inside_last) % UINT256_MAX) // (2**128)
    return owed
```

### When to Collect

| Trigger | Action |
|---------|--------|
| Fees > 0.5% of position value | Collect and redeploy |
| About to rebalance range | Always collect first |
| Token price spike | Collect immediately — fees surge |
| Gas cost > 10% of fees owed | Wait — collecting isn't worth it yet |

---

## 1E. Deposits

### Choosing a Range

Range selection is the highest-impact decision in LP management.

```python
def suggest_range(current_price, volatility_daily, target_days_in_range=14,
                  fee_tier=0.003):
    """
    Suggest a tick range that keeps you in-range for ~target_days_in_range.
    Uses 1-sigma daily vol scaled to the target window.
    """
    import math
    # 1-sigma price range over N days
    sigma_n = volatility_daily * math.sqrt(target_days_in_range)
    p_low  = current_price * math.exp(-sigma_n)
    p_high = current_price * math.exp(+sigma_n)

    # Convert to ticks (tick = log(price) / log(1.0001))
    tick_lower = math.floor(math.log(p_low)  / math.log(1.0001))
    tick_upper = math.ceil( math.log(p_high) / math.log(1.0001))

    # Snap to tick spacing (0.05% = 10, 0.3% = 60, 1% = 200)
    spacing = {0.0001: 1, 0.0005: 10, 0.003: 60, 0.01: 200}[fee_tier]
    tick_lower = (tick_lower // spacing) * spacing
    tick_upper = ((tick_upper // spacing) + 1) * spacing

    return {
        "price_lower": p_low,
        "price_upper": p_high,
        "tick_lower": tick_lower,
        "tick_upper": tick_upper,
        "range_width_pct": (p_high / p_low - 1) * 100,
        "estimated_days_in_range": target_days_in_range,
    }
```

### Computing Token Amounts from Liquidity L

```python
def liquidity_to_amounts(L, current_price, p_low, p_high):
    """
    Given liquidity L and range, compute token0 and token1 amounts.
    """
    import math
    sp  = math.sqrt(current_price)
    spa = math.sqrt(p_low)
    spb = math.sqrt(p_high)

    if current_price <= p_low:
        # All token0
        amount0 = L * (1/spa - 1/spb)
        amount1 = 0.0
    elif current_price >= p_high:
        # All token1
        amount0 = 0.0
        amount1 = L * (spb - spa)
    else:
        # Mixed
        amount0 = L * (1/sp - 1/spb)
        amount1 = L * (sp - spa)

    return amount0, amount1

def amounts_to_liquidity(amount0, amount1, current_price, p_low, p_high):
    """
    Given token amounts and range, compute the resulting L.
    Returns L and the actual amounts used (one may be limiting factor).
    """
    import math
    sp  = math.sqrt(current_price)
    spa = math.sqrt(p_low)
    spb = math.sqrt(p_high)

    if current_price <= p_low:
        L = amount0 / (1/spa - 1/spb)
    elif current_price >= p_high:
        L = amount1 / (spb - spa)
    else:
        L0 = amount0 / (1/sp - 1/spb)
        L1 = amount1 / (sp - spa)
        L  = min(L0, L1)          # binding constraint sets L

    a0, a1 = liquidity_to_amounts(L, current_price, p_low, p_high)
    return L, a0, a1
```

---

## 1F. Withdrawals

Withdrawal is **not just the reverse of deposit**. When you withdraw you are
crystallising whatever IL has occurred. The process:

```
1. Collect all owed fees FIRST (separate transaction)
2. Remove liquidity (decreaseLiquidity) → tokens returned at current ratio
3. Record: token0_received, token1_received, fees0_total, fees1_total
4. Compute P&L (see Phase 2)
```

### Withdrawal Output Tracking

```python
@dataclass
class WithdrawalRecord:
    position_id:      str
    close_timestamp:  int
    close_price:      float

    # Tokens received from liquidity removal
    token0_from_liq:  float
    token1_from_liq:  float

    # Fees collected (total over entire position lifetime)
    fees0_total:      float
    fees1_total:      float

    # USD values at close prices
    liq_value_usd:    float   # token0_from_liq + token1_from_liq in USD
    fees_value_usd:   float   # fees in USD at close price

    # Computed P&L fields (filled in Phase 2)
    il_usd:           float = 0.0
    onl_usd:          float = 0.0
    net_profit_usd:   float = 0.0
    fee_apr_realised: float = 0.0
    days_held:        int   = 0
```

---

## 1G. Rebalancing

Rebalancing means **closing the current position and opening a new one** centred
on the current price. It is triggered when price has drifted too far from your
range midpoint.

### Rebalance Trigger Rules

```python
def should_rebalance(position, current_price, config):
    """Returns (bool, reason)"""

    # 1. Price left the range entirely
    if current_price < position.p_low or current_price > position.p_high:
        return True, "OUT_OF_RANGE"

    # 2. Price in bottom 20% or top 20% of range (about to exit)
    range_width = position.p_high - position.p_low
    lower_band  = position.p_low  + 0.20 * range_width
    upper_band  = position.p_high - 0.20 * range_width
    if current_price < lower_band or current_price > upper_band:
        return True, "NEAR_RANGE_EDGE"

    # 3. Time-based: position older than max hold period
    days_held = (time.time() - position.entry_timestamp) / 86400
    if days_held > config.max_hold_days:
        return True, "MAX_HOLD_EXCEEDED"

    # 4. Fees owed > rebalance_fee_threshold (harvest and recenter)
    if position.fees_owed_usd > config.rebalance_fee_threshold_usd:
        return True, "FEE_HARVEST"

    return False, None
```

### Rebalance Swap Logic

After withdrawing, you will usually have the wrong token ratio for the new range.
You must swap to rebalance your inventory before depositing.

```python
def compute_swap_for_rebalance(token0_held, token1_held, token0_price_in_token1,
                                target_ratio_token0):
    """
    target_ratio_token0: fraction of value that should be in token0 (0.0–1.0)
    Returns: (token_to_sell, amount_to_sell)
    """
    total_value_in_token1 = token0_held * token0_price_in_token1 + token1_held
    target_value0 = total_value_in_token1 * target_ratio_token0
    current_value0 = token0_held * token0_price_in_token1

    delta = current_value0 - target_value0

    if delta > 0:
        # Sell token0 to get token1
        return "SELL_TOKEN0", delta / token0_price_in_token1
    elif delta < 0:
        # Sell token1 to get token0
        return "SELL_TOKEN1", abs(delta)
    else:
        return "NO_SWAP", 0.0

def target_ratio_for_range(current_price, p_low, p_high):
    """What fraction of deposit value should be in token0 for this range?"""
    import math
    sp  = math.sqrt(current_price)
    spa = math.sqrt(p_low)
    spb = math.sqrt(p_high)
    if current_price <= p_low:  return 1.0
    if current_price >= p_high: return 0.0
    val0 = sp - spa
    val1 = (1/sp - 1/spb) * current_price
    return val0 / (val0 + val1)
```

---

## 1H. Inventory Management

Inventory = the token balances you are managing across all open positions plus
your undeployed wallet balance.

```python
@dataclass
class Inventory:
    # Wallet (undeployed)
    wallet_token0: float
    wallet_token1: float

    # Deployed in LP positions
    positions: list   # list of PositionState

    # Pending fees (owed but not collected)
    pending_fees0: float
    pending_fees1: float

    # Reference prices
    token0_price_usd: float
    token1_price_usd: float

    @property
    def total_token0(self):
        in_positions = sum(p.token0_held for p in self.positions)
        return self.wallet_token0 + in_positions + self.pending_fees0

    @property
    def total_token1(self):
        in_positions = sum(p.token1_held for p in self.positions)
        return self.wallet_token1 + in_positions + self.pending_fees1

    @property
    def total_value_usd(self):
        return (self.total_token0 * self.token0_price_usd +
                self.total_token1 * self.token1_price_usd)

    @property
    def token0_exposure_pct(self):
        return (self.total_token0 * self.token0_price_usd) / self.total_value_usd
```

### Inventory Risk Rule

```python
def inventory_risk_check(inventory, config):
    """
    Returns list of risk flags.
    config.max_token0_exposure: e.g. 0.70 (no more than 70% in token0)
    config.min_undeployed_pct:  e.g. 0.05 (keep 5% as gas buffer)
    """
    flags = []
    if inventory.token0_exposure_pct > config.max_token0_exposure:
        flags.append(f"OVEREXPOSED_TOKEN0: {inventory.token0_exposure_pct:.1%}")
    undeployed = (inventory.wallet_token0 * inventory.token0_price_usd +
                  inventory.wallet_token1 * inventory.token1_price_usd)
    if undeployed / inventory.total_value_usd < config.min_undeployed_pct:
        flags.append("INSUFFICIENT_GAS_BUFFER")
    return flags
```

---

## 1I. True Net Profit Calculation

This is the final answer after a position closes. All four components together:

```python
def calculate_net_profit(position, withdrawal, token0_price_usd,
                         benchmark_value_at_close):
    """
    Full P&L decomposition for a closed position.
    Returns a dict with all four P&L components.
    """
    hold_duration_days = withdrawal.days_held

    # 1. Fee income (in USD)
    fee_income_usd = withdrawal.fees_value_usd

    # 2. LP position value at close (tokens at current price)
    lp_close_value_usd = withdrawal.liq_value_usd

    # 3. HODL value (what same tokens would be worth if never deposited)
    hodl_value_usd = (position.initial_token0 * token0_price_usd +
                      position.initial_token1)  # token1 = stablecoin assumed

    # 4. IL = LP close value - HODL value (negative = loss vs hodl)
    il_usd = lp_close_value_usd - hodl_value_usd

    # 5. ONL = total LP return - benchmark return (negative = you underperformed)
    total_lp_value = lp_close_value_usd + fee_income_usd
    onl_usd = total_lp_value - benchmark_value_at_close

    # 6. Net profit vs initial investment
    net_profit_usd = total_lp_value - position.entry_value_usd

    # 7. Fee APR (annualised)
    fee_apr = (fee_income_usd / position.entry_value_usd) * (365 / hold_duration_days) * 100

    return {
        "entry_value_usd":    round(position.entry_value_usd, 2),
        "fee_income_usd":     round(fee_income_usd, 2),
        "lp_close_value_usd": round(lp_close_value_usd, 2),
        "hodl_value_usd":     round(hodl_value_usd, 2),
        "il_usd":             round(il_usd, 2),
        "il_pct":             round(il_usd / hodl_value_usd * 100, 3),
        "onl_usd":            round(onl_usd, 2),
        "total_lp_value_usd": round(total_lp_value, 2),
        "net_profit_usd":     round(net_profit_usd, 2),
        "net_profit_pct":     round(net_profit_usd / position.entry_value_usd * 100, 3),
        "fee_apr_realised":   round(fee_apr, 2),
        "days_held":          hold_duration_days,
        "verdict":            "BEAT_HODL" if (fee_income_usd + il_usd) > 0 else "LOST_TO_HODL",
        "beat_benchmark":     onl_usd > 0,
    }
```

---

# PHASE 2 — BACKTEST

## Objective

Simulate the full LP lifecycle — deposit, hold, collect fees, rebalance when
triggered, withdraw — over historical pool data. Measure true net profit,
IL, ONL, and fee income for different range strategies.

## Strategies to Test

Run all four in parallel on the same pool universe and date range:

| Strategy | Range Method | Rebalance Trigger |
|----------|-------------|------------------|
| A — Narrow | ±5% around current price | Out-of-range only |
| B — Medium | 1-sigma 14-day volatility | Out-of-range or near edge |
| C — Wide | 2-sigma 14-day volatility | Monthly, time-based |
| D — Full Range | Min/max ticks (v2-style) | Never |

## Backtest Engine

```python
def backtest_lp_strategy(pool_daily_history, strategy_config, capital_usd=10000):
    """
    pool_daily_history: list of { date, open, high, low, close,
                                   feesUSD, tvlUSD, volumeUSD }
    strategy_config: { range_method, rebalance_trigger, fee_tier }
    Returns: list of trade records + portfolio summary
    """
    trades        = []
    current_pos   = None
    capital       = capital_usd
    fees_collected = 0.0

    for i, day in enumerate(pool_daily_history):
        price = day['close']
        date  = day['date']

        # --- OPEN POSITION ---
        if current_pos is None:
            vol_30d = daily_vol(pool_daily_history, i, 30)
            rng = suggest_range(price, vol_30d,
                                strategy_config['target_days'],
                                strategy_config['fee_tier'])
            current_pos = {
                "entry_date":    date,
                "entry_price":   price,
                "entry_capital": capital,
                "p_low":  rng['price_lower'],
                "p_high": rng['price_upper'],
                "fees_accumulated": 0.0,
            }
            continue

        # --- ACCRUE FEES ---
        # Fees earned proportional to position's share of pool liquidity
        # Approximated as: (position_capital / pool_tvl) * pool_feesUSD
        pos_share = capital / day['tvlUSD'] if day['tvlUSD'] > 0 else 0
        in_range  = current_pos['p_low'] <= price <= current_pos['p_high']
        day_fees  = pos_share * day['feesUSD'] * (1.0 if in_range else 0.0)
        current_pos['fees_accumulated'] += day_fees

        # --- CHECK REBALANCE ---
        trigger, reason = should_rebalance_simple(
            current_pos, price, strategy_config
        )

        if trigger:
            # CLOSE current position
            il_frac = il_full_range(price / current_pos['entry_price'])
            close_lp_value = capital * (1 + il_frac)
            fees_usd       = current_pos['fees_accumulated']
            total_value    = close_lp_value + fees_usd
            hodl_value     = capital  # stablecoin pair assumed

            trades.append({
                "open_date":    current_pos['entry_date'],
                "close_date":   date,
                "days_held":    (date - current_pos['entry_date']) // 86400,
                "strategy":     strategy_config['name'],
                "entry_capital": capital,
                "close_lp_usd": round(close_lp_value, 2),
                "fees_usd":     round(fees_usd, 2),
                "total_usd":    round(total_value, 2),
                "il_usd":       round(close_lp_value - hodl_value, 2),
                "net_profit_usd": round(total_value - capital, 2),
                "rebalance_reason": reason,
            })

            capital        = total_value
            fees_collected += fees_usd
            current_pos    = None  # will reopen next day

    return trades

def daily_vol(history, idx, window):
    """Annualised daily vol from log returns over window days."""
    import numpy as np, math
    start = max(0, idx - window)
    closes = [d['close'] for d in history[start:idx+1]]
    if len(closes) < 2: return 0.05
    returns = [math.log(closes[i]/closes[i-1]) for i in range(1, len(closes))]
    return np.std(returns) * (252 ** 0.5)
```

## Backtest Metrics to Compute

```python
def summarise_trades(trades, initial_capital):
    import numpy as np

    net_profits = [t['net_profit_usd'] for t in trades]
    fee_incomes = [t['fees_usd'] for t in trades]
    il_values   = [t['il_usd'] for t in trades]

    total_return = sum(net_profits) / initial_capital * 100
    fee_return   = sum(fee_incomes) / initial_capital * 100
    il_total     = sum(il_values)

    trade_returns = [t['net_profit_usd'] / t['entry_capital'] for t in trades]
    sharpe = np.mean(trade_returns) / (np.std(trade_returns) + 1e-9) * (252 ** 0.5)

    return {
        "strategy":         trades[0]['strategy'] if trades else "unknown",
        "num_rebalances":   len(trades),
        "total_return_pct": round(total_return, 2),
        "fee_return_pct":   round(fee_return, 2),
        "il_total_usd":     round(il_total, 2),
        "sharpe":           round(sharpe, 3),
        "pct_beat_hodl":    round(sum(1 for t in trades if t['net_profit_usd'] > 0) / len(trades) * 100, 1),
        "avg_days_per_range": round(np.mean([t['days_held'] for t in trades]), 1),
    }
```

## Backtest Output Format

```json
{
  "workshop": "02",
  "phase": "backtest",
  "pool": "WETH/USDC 0.05%",
  "period": "2025-06-01 to 2026-02-01",
  "initial_capital_usd": 10000,
  "strategies": {
    "A_narrow":     { "num_rebalances": 28, "total_return_pct": 14.2, "fee_return_pct": 19.1, "il_total_usd": -487, "sharpe": 0.71 },
    "B_medium":     { "num_rebalances": 11, "total_return_pct": 11.3, "fee_return_pct": 13.4, "il_total_usd": -204, "sharpe": 0.84 },
    "C_wide":       { "num_rebalances": 4,  "total_return_pct": 8.7,  "fee_return_pct": 8.9,  "il_total_usd": -18,  "sharpe": 0.92 },
    "D_full_range": { "num_rebalances": 0,  "total_return_pct": 3.1,  "fee_return_pct": 4.8,  "il_total_usd": -171, "sharpe": 0.41 }
  },
  "winner": "C_wide",
  "key_finding": "Gas costs erode narrow strategy returns. Medium/wide outperform on risk-adjusted basis."
}
```

Save as `evals/backtest_results_workshop02.json`.

---

# PHASE 3 — EXECUTE

## The Position Management Loop

This is the live operational loop. Run it once per day (or per event).

```
DAILY LOOP:
  For each open position:
    1. Query current price from subgraph or RPC
    2. Query owed fees (compute_fees_owed)
    3. Update PositionState (token amounts, IL, ONL)
    4. Check rebalance triggers
    5. If collect_trigger: execute collectFees()
    6. If rebalance_trigger: execute rebalance sequence
    7. Log daily snapshot to STATE.md
```

## Live Position Snapshot Query

```graphql
query LivePosition($positionId: ID!) {
  position(id: $positionId) {
    id
    owner
    liquidity
    depositedToken0
    depositedToken1
    withdrawnToken0
    withdrawnToken1
    collectedFeesToken0
    collectedFeesToken1
    feeGrowthInside0LastX128
    feeGrowthInside1LastX128
    tickLower { tickIdx price0 price1 feeGrowthOutside0X128 feeGrowthOutside1X128 }
    tickUpper { tickIdx price0 price1 feeGrowthOutside0X128 feeGrowthOutside1X128 }
    pool {
      sqrtPrice
      tick
      token0Price
      token1Price
      feeGrowthGlobal0X128
      feeGrowthGlobal1X128
      totalValueLockedUSD
      volumeUSD
      feesUSD
    }
    transaction { timestamp }
  }
}
```

## Daily P&L Snapshot Output

```json
{
  "snapshot_date":     "2026-02-17",
  "position_id":       "12345",
  "pool":              "WETH/USDC 0.05%",
  "entry_date":        "2026-01-20",
  "days_held":         28,

  "entry_price":       3100.00,
  "current_price":     3210.45,
  "range_lower":       2900.00,
  "range_upper":       3350.00,
  "in_range":          true,

  "token0_held":       1.21,
  "token1_held":       1124.50,
  "fees0_owed":        0.012,
  "fees1_owed":        38.20,

  "entry_value_usd":   10000.00,
  "current_lp_usd":    10014.80,
  "fees_value_usd":    76.70,
  "total_value_usd":   10091.50,

  "hodl_value_usd":    10218.30,
  "il_usd":            -203.50,
  "il_pct":            -1.99,

  "benchmark_value_usd": 10350.00,
  "onl_usd":           -258.50,

  "net_profit_usd":    91.50,
  "net_profit_pct":    0.915,
  "fee_apr_annualised": 10.0,

  "rebalance_check": {
    "should_rebalance": false,
    "reason": null,
    "price_position_in_range": "48%"
  }
}
```

## Rebalance Execution Sequence

When a rebalance is triggered, execute in this exact order. Order matters —
wrong sequence leads to failed transactions or suboptimal fills.

```
Step 1  collectFees()        → realise all owed fees to wallet
Step 2  decreaseLiquidity()  → remove all liquidity, get tokens back
Step 3  compute new range    → suggest_range(current_price, vol_30d)
Step 4  compute_swap_for_rebalance() → determine which token to sell and how much
Step 5  execute swap         → via Trading API or direct router call
Step 6  increaseLiquidity() / mint() → open new position at new range
Step 7  log rebalance record → save to evals/rebalance_log_workshop02.json
```

## Rebalance Log Format

```json
{
  "workshop": "02",
  "phase": "execute",
  "timestamp": "2026-02-17T14:32:00Z",
  "trigger":   "NEAR_RANGE_EDGE",

  "before": {
    "price":        3210.45,
    "range":        [2900, 3350],
    "lp_value_usd": 10014.80,
    "fees_usd":     76.70,
    "il_usd":       -203.50
  },

  "swap": {
    "action":        "SELL_TOKEN0",
    "amount_in":     0.18,
    "amount_in_usd": 578.0,
    "amount_out":    577.4,
    "slippage_pct":  0.10,
    "gas_usd":       4.20
  },

  "after": {
    "price":         3210.45,
    "new_range":     [3050, 3390],
    "lp_value_usd":  10087.00,
    "total_cost_usd": 4.20
  },

  "cumulative_rebalances": 3,
  "cumulative_gas_cost_usd": 13.80
}
```

---

# PHASE 4 — COLLECT RESULTS

## 30-Day Position Audit

At the end of every 30 days (or on position close), run the full P&L audit:

```python
def full_position_audit(position_id, subgraph_endpoint, benchmark_prices):
    """
    Fetch all historical data for a position and compute final P&L.
    """
    # 1. Get position record from subgraph
    pos = fetch_position(position_id, subgraph_endpoint)

    # 2. Get pool daily data over holding period
    history = fetch_pool_day_data(pos['pool']['id'],
                                  pos['entry_timestamp'],
                                  int(time.time()),
                                  subgraph_endpoint)

    # 3. Compute metrics
    entry_price   = float(pos['pool']['token0Price_at_entry'])
    current_price = float(pos['pool']['token0Price'])
    days_held     = (time.time() - pos['entry_timestamp']) / 86400

    # Total fees (collected + owed)
    fees0 = float(pos['collectedFeesToken0']) + pos['fees0_owed']
    fees1 = float(pos['collectedFeesToken1']) + pos['fees1_owed']
    fees_usd = fees0 * current_price + fees1

    # Entry value
    entry_value = (float(pos['depositedToken0']) * entry_price +
                   float(pos['depositedToken1']))

    # Current LP value
    a0, a1 = liquidity_to_amounts(int(pos['liquidity']), current_price,
                                   pos['p_low'], pos['p_high'])
    lp_value_usd = a0 * current_price + a1

    # HODL value
    hodl_value_usd = (float(pos['depositedToken0']) * current_price +
                      float(pos['depositedToken1']))

    # ONL vs benchmark (e.g. Aave USDC yield at 4% APY)
    aave_apy   = 0.04
    benchmark_value = entry_value * (1 + aave_apy * days_held / 365)

    return calculate_net_profit(
        position  = SimpleNamespace(
            initial_token0=float(pos['depositedToken0']),
            initial_token1=float(pos['depositedToken1']),
            entry_value_usd=entry_value,
        ),
        withdrawal = SimpleNamespace(
            liq_value_usd = lp_value_usd,
            fees_value_usd = fees_usd,
            days_held = int(days_held),
        ),
        token0_price_usd      = current_price,
        benchmark_value_at_close = benchmark_value,
    )
```

## Workshop 02 Results Format

```json
{
  "workshop":    "02",
  "phase":       "collect",
  "date":        "2026-03-19",
  "positions_audited": 3,

  "results": [
    {
      "position_id":       "12345",
      "pool":              "WETH/USDC 0.05%",
      "strategy":          "B_medium",
      "days_held":         30,
      "num_rebalances":    2,
      "gas_cost_usd":      9.20,

      "entry_value_usd":   10000.00,
      "fee_income_usd":    142.30,
      "lp_close_value_usd": 9831.20,
      "hodl_value_usd":    10218.30,

      "il_usd":            -387.10,
      "il_pct":            -3.79,
      "fees_covered_il_pct": 36.7,

      "onl_usd":           -276.80,
      "onl_benchmark":     "Aave USDC 4% APY",

      "net_profit_usd":    -26.50,
      "net_profit_pct":    -0.265,
      "fee_apr_realised":  17.1,

      "verdict":           "LOST_TO_HODL",
      "beat_benchmark":    false,
      "lessons":           "IL exceeded fee income. Market moved 12% in 30d. Need wider range or higher fee tier pool."
    }
  ],

  "portfolio_summary": {
    "total_net_profit_usd":   -41.20,
    "total_fee_income_usd":   398.60,
    "total_il_usd":           -831.40,
    "total_gas_cost_usd":     27.80,
    "avg_fee_apr":             15.4,
    "strategies_beat_hodl":    1,
    "strategies_beat_benchmark": 0
  },

  "strategy_update": {
    "winner":     "C_wide",
    "finding":    "Wide ranges outperformed in volatile market. Narrow strategy gas costs erased fee gains.",
    "next_range_method": "2-sigma 14-day vol",
    "rebalance_trigger_update": "Raise near-edge threshold from 20% to 30% of range"
  }
}
```

---

## Closing the Loop

### What to Write in STATE.md After This Workshop

```markdown
## WORKSHOP_02 — Completed [DATE]

### Range Strategy Winner
Strategy [A/B/C/D]: [reason]
New default range method: [sigma × days]

### P&L Decomposition Findings
- Avg fee income USD: [value]
- Avg IL USD: [value]
- Fees covered IL: [pct]%
- % positions beat HODL: [pct]%
- % positions beat benchmark: [pct]%

### Rebalance Parameter Updates
- Trigger threshold: [updated value]
- Max hold days: [updated value]
- Fee collect threshold: [updated value]
- Gas cost must be < [pct]% of fees owed before collecting

### ONL Benchmark Selected
[HODL_50_50 / HODL_100_ETH / Aave_APY / other]
Reason: [why this benchmark is most relevant]

### Key Lessons
- [lesson 1]
- [lesson 2]
- [lesson 3]

### WORKSHOP_02: SCHEMA_NOTES
- [any subgraph field surprises]
```

---

## Workshop Progression

```
Workshop 01 ✓
  Signals, factor models, pool ranking, LP allocation

Workshop 02 (this file)
  Deposits, withdrawals, fees, rebalancing, IL, ONL, net profit

        │
        ▼
Workshop 03
  Multi-position portfolio: correlations, hedge ratios,
  cross-pool rebalancing, delta-neutral LP construction,
  gas optimisation (batching), automated monitoring alerts

        │
        ▼
Workshop 04
  On-chain arb detection: v2/v3/v4 cross-pool spreads,
  sandwich attack awareness, MEV protection patterns,
  JIT liquidity detection, toxic flow identification
```

---

## Files to Commit After This Workshop

```
evals/backtest_results_workshop02.json     ← strategy comparison
evals/rebalance_log_workshop02.json        ← all rebalance events
evals/results_workshop02_YYYYMMDD.json     ← 30-day position audit
STATE.md                                   ← updated with WORKSHOP_02 notes
```

---

*UniClaw Workshop 02 | https://github.com/ylintern/uniclawskills | MIT License*
