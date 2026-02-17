# UniClaw Workshop 03 — Edge, Arbitrage & Capital Safety
### Learn → Backtest → Execute → Collect

> **Prerequisites:** Workshops 01 and 02 complete. Range strategy winner
> recorded in `STATE.md`. P&L decomposition formulas internalised.
>
> **What this workshop trains:** Finding and capturing edges — price arbitrage,
> flash swaps, tick-zone gaps, single-side LP, cross-chain spreads — while
> building a gas-profit prediction model that filters out unprofitable ops
> before they execute. The safety framework runs above every decision.
> Capital preservation is the primary constraint. Edge is only taken when
> the risk-adjusted math clears. When in doubt, do nothing.
>
> **Tag all STATE.md entries this session:** `WORKSHOP_03:`

---

## Before You Start — The Edge Operator's Hierarchy

```
HIERARCHY (read top-down, never skip a level):

  1. CAPITAL SAFETY          — Is the operation recoverable if it fails?
  2. GAS-PROFIT VIABILITY    — Does the math clear after all costs?
  3. EXECUTION RISK          — Can it be executed atomically or safely staged?
  4. EDGE CAPTURE            — Only now do you take the trade.
```

Every strategy in this workshop passes through all four gates before execution.
A beautiful arb opportunity that fails gate 1 is not an opportunity — it is a trap.

---

# PHASE 1 — LEARN

---

## 1A. Capital Safety Framework

This section is not optional. Read it before any other section in this workshop.

### The Five Safety Rules

```
RULE 1 — NEVER RISK MORE THAN YOU CAN LOSE COMPLETELY
  Flash swaps, arb, and cross-chain ops can fail at the execution layer.
  Size every operation so that a total loss of that capital does not
  impair your ability to continue operating.
  Max single operation size: 5% of total portfolio.

RULE 2 — ATOMIC OR STAGED, NEVER EXPOSED
  Every multi-step operation must be either:
  (a) Atomic — fully on-chain in one transaction that reverts if any step fails, OR
  (b) Staged — each step is safe to hold indefinitely if the next step fails.
  Never leave yourself in a state where step 1 has executed but step 2
  is pending and you are exposed to price risk on an unhedged position.

RULE 3 — SIMULATE BEFORE SUBMIT
  Use eth_call (static call) to simulate the full transaction before
  broadcasting. If simulation reverts, do not submit. Gas cost of a
  reverted transaction is lost but capital is preserved.

RULE 4 — SLIPPAGE CAPS ARE HARD LIMITS
  Set amountOutMinimum in every swap. If the market moves past your
  slippage cap before execution, the transaction reverts. Never remove
  or inflate slippage caps to force a trade through.

RULE 5 — GAS PRICE CEILING
  Set a maxFeePerGas on every transaction. If the gas market is above
  your ceiling, do not submit. Wait. A missed trade costs nothing.
  A trade executed at 10× expected gas cost destroys the P&L.
```

### Safety State Machine

```python
class OperationSafety:
    """Gate every operation through this before execution."""

    def __init__(self, portfolio_value_usd, config):
        self.portfolio  = portfolio_value_usd
        self.config     = config   # max_op_pct, max_gas_usd, max_slippage_pct

    def gate(self, op):
        """
        op: { type, capital_at_risk_usd, expected_profit_usd,
              gas_estimate_usd, slippage_pct, is_atomic }
        Returns: (approved: bool, reason: str)
        """
        # Gate 1 — size
        if op['capital_at_risk_usd'] > self.portfolio * self.config['max_op_pct']:
            return False, f"SIZE_EXCEEDS_LIMIT: {op['capital_at_risk_usd']:.0f} > {self.portfolio * self.config['max_op_pct']:.0f}"

        # Gate 2 — gas viability
        net = op['expected_profit_usd'] - op['gas_estimate_usd']
        if net <= 0:
            return False, f"GAS_EXCEEDS_PROFIT: net={net:.2f}"

        # Gate 3 — minimum profit threshold (noise filter)
        if net < self.config['min_profit_usd']:
            return False, f"BELOW_MIN_PROFIT: {net:.2f} < {self.config['min_profit_usd']}"

        # Gate 4 — slippage
        if op['slippage_pct'] > self.config['max_slippage_pct']:
            return False, f"SLIPPAGE_TOO_HIGH: {op['slippage_pct']:.2f}%"

        # Gate 5 — atomicity
        if not op['is_atomic'] and op['type'] in ['flash_swap','arb']:
            return False, "NON_ATOMIC_ARB_FORBIDDEN"

        return True, "APPROVED"
```

---

## 1B. Gas Cost vs Profit Prediction

Gas prediction is not optional — it is the primary filter that eliminates
95% of apparent opportunities before they waste capital.

### Gas Cost Components

```
TOTAL COST = (gas_units × base_fee + gas_units × priority_fee) / 1e9 × ETH_price_usd
           + any protocol fees (trading fee, bridge fee)
           + slippage cost estimate
```

```python
async def estimate_gas_cost_usd(tx_data, web3, eth_price_usd):
    """
    Simulate transaction, get gas estimate, price it in USD.
    Returns: { gas_units, base_fee_gwei, priority_fee_gwei,
               gas_cost_eth, gas_cost_usd, safe_to_submit }
    """
    # 1. Simulate (will revert if tx would fail — catches errors before spending gas)
    try:
        gas_units = web3.eth.estimate_gas(tx_data)
    except Exception as e:
        return { "safe_to_submit": False, "revert_reason": str(e) }

    # 2. Current fee market
    fee_history = web3.eth.fee_history(5, 'latest', [25, 50, 75])
    base_fee    = fee_history['baseFeePerGas'][-1] / 1e9        # gwei
    priority_25 = fee_history['reward'][-1][0] / 1e9            # conservative tip

    # 3. Cost
    total_gwei  = base_fee + priority_25
    gas_eth     = gas_units * total_gwei / 1e9
    gas_usd     = gas_eth * eth_price_usd

    # 4. Safety buffer: add 20% margin for gas market moves
    gas_usd_buffered = gas_usd * 1.20

    return {
        "gas_units":       gas_units,
        "base_fee_gwei":   round(base_fee, 2),
        "priority_gwei":   round(priority_25, 2),
        "gas_cost_eth":    round(gas_eth, 6),
        "gas_cost_usd":    round(gas_usd, 2),
        "gas_cost_usd_buffered": round(gas_usd_buffered, 2),
        "safe_to_submit":  True
    }

# Operation type gas benchmarks (approximate, Ethereum mainnet)
GAS_BENCHMARKS = {
    "erc20_transfer":          50_000,
    "uniswap_v3_swap":        130_000,
    "uniswap_v3_mint":        200_000,
    "uniswap_v3_collect":      80_000,
    "uniswap_v3_burn":         90_000,
    "uniswap_v4_swap":         90_000,   # cheaper due to singleton
    "uniswap_v4_mint":        160_000,
    "flash_swap_v3":          200_000,   # base + callback logic
    "flash_swap_v3_with_arb": 300_000,   # base + swap in callback
    "cross_chain_bridge":     250_000,   # varies wildly by bridge
}

def min_profit_to_breakeven(op_type, eth_price_usd, gas_gwei=30,
                             safety_multiplier=1.5):
    """
    Minimum gross profit (USD) required for this op type to be worth running.
    safety_multiplier: extra cushion above pure breakeven.
    """
    gas_units = GAS_BENCHMARKS.get(op_type, 200_000)
    gas_cost  = gas_units * gas_gwei / 1e9 * eth_price_usd
    return gas_cost * safety_multiplier
```

### Gas Prediction Model (Historical Regression)

```python
import numpy as np
from datetime import datetime

class GasPricePredictor:
    """
    Predict likely gas price for a target execution window
    using historical base fee data.
    """
    # Historical pattern: gas is typically lowest between 00:00–06:00 UTC
    # and highest between 13:00–20:00 UTC (US trading hours overlap EU)
    HOURLY_MULTIPLIERS = {
        0: 0.65, 1: 0.60, 2: 0.58, 3: 0.57, 4: 0.59, 5: 0.63,
        6: 0.72, 7: 0.81, 8: 0.90, 9: 0.96, 10: 1.00, 11: 1.03,
        12: 1.05, 13: 1.08, 14: 1.10, 15: 1.10, 16: 1.09, 17: 1.07,
        18: 1.04, 19: 1.00, 20: 0.95, 21: 0.88, 22: 0.79, 23: 0.71,
    }

    def predict_gas(self, base_gwei_now, target_hour_utc):
        current_hour = datetime.utcnow().hour
        scale = self.HOURLY_MULTIPLIERS[target_hour_utc] / \
                self.HOURLY_MULTIPLIERS[current_hour]
        return base_gwei_now * scale

    def best_execution_window(self, base_gwei_now, max_acceptable_gwei):
        """Return list of UTC hours where predicted gas is below threshold."""
        return [h for h in range(24)
                if self.predict_gas(base_gwei_now, h) <= max_acceptable_gwei]
```

---

## 1C. Price Arbitrage

### Types of Arb Available on Uniswap

```
TYPE 1 — SAME PAIR, DIFFERENT FEE TIER (most common)
  WETH/USDC at 0.05% and 0.3% quote different prices.
  Buy on cheap pool, sell on expensive pool.

TYPE 2 — SAME PAIR, DIFFERENT VERSION (v2 vs v3 vs v4)
  v2 WETH/USDC vs v3 WETH/USDC. Price lag between versions.

TYPE 3 — TRIANGULAR (three tokens, one path)
  ETH → USDC → WBTC → ETH. Exploit circular mispricing.

TYPE 4 — CROSS-DEX (Uniswap vs Curve vs Balancer)
  Same asset, different AMM — price discrepancy.

TYPE 5 — TICK ARBITRAGE (covered in 1F)
  Exploit price gaps in the tick liquidity distribution.
```

### Simple Two-Pool Arb

```python
def find_two_pool_arb(pool_a_price, pool_b_price,
                      pool_a_fee, pool_b_fee,
                      amount_in_usd, eth_price_usd, gas_gwei=30):
    """
    Check if a buy-on-A, sell-on-B arb clears costs.
    Prices: token1 per token0 (e.g. USDC per ETH).
    Fees: e.g. 0.003 for 0.3%.
    """
    if pool_a_price >= pool_b_price:
        return None   # no spread in this direction

    spread_raw = (pool_b_price - pool_a_price) / pool_a_price

    # Net spread after both fees
    net_spread = spread_raw - pool_a_fee - pool_b_fee

    if net_spread <= 0:
        return None   # fees eat the entire spread

    gross_profit_usd = amount_in_usd * net_spread

    # Gas cost for 2 swaps
    gas_cost_usd = (GAS_BENCHMARKS['uniswap_v3_swap'] * 2 *
                    gas_gwei / 1e9 * eth_price_usd)

    net_profit_usd = gross_profit_usd - gas_cost_usd

    return {
        "direction":        "BUY_A_SELL_B",
        "spread_raw_pct":   round(spread_raw * 100, 4),
        "net_spread_pct":   round(net_spread * 100, 4),
        "gross_profit_usd": round(gross_profit_usd, 2),
        "gas_cost_usd":     round(gas_cost_usd, 2),
        "net_profit_usd":   round(net_profit_usd, 2),
        "viable":           net_profit_usd > 0
    }
```

### Triangular Arbitrage Path Scanner

```python
def check_triangular_arb(prices, fees, path, amount_in):
    """
    prices: dict of "TOKEN_A/TOKEN_B" → price
    fees:   dict of "TOKEN_A/TOKEN_B" → fee rate
    path:   ["ETH", "USDC", "WBTC", "ETH"]
    amount_in: starting amount of path[0] token
    Returns: (profit_pct, profitable: bool)
    """
    amount = amount_in
    for i in range(len(path) - 1):
        a, b   = path[i], path[i+1]
        key    = f"{a}/{b}"
        rkey   = f"{b}/{a}"
        if key in prices:
            fee    = fees.get(key, 0.003)
            amount = amount * prices[key] * (1 - fee)
        elif rkey in prices:
            fee    = fees.get(rkey, 0.003)
            amount = amount / prices[rkey] * (1 - fee)
        else:
            return 0, False

    profit_pct = (amount - amount_in) / amount_in * 100
    return round(profit_pct, 4), profit_pct > 0
```

---

## 1D. Flash Swaps

### What a Flash Swap Is

A flash swap lets you borrow any amount of any token from a Uniswap pool,
use it within a single transaction, and return it (plus a fee) — all atomically.
If the callback cannot return the tokens, the entire transaction reverts.
**You never risk losing the borrowed principal** — the worst outcome is
the transaction reverts and you pay the gas.

```
Normal swap:     you send token IN → pool → you receive token OUT
Flash swap:      pool → you receive token OUT → (callback executes) → you return token IN + fee
```

### Flash Swap Arb Pattern

```
1. Borrow 100 ETH from Uniswap v3 pool A (fee: 0.05%)
2. In callback:
   a. Sell 100 ETH on pool B (v2) → receive more USDC than needed
   b. Buy back required ETH on pool A to repay + fee
   c. Pocket the USDC surplus
3. Transaction reverts if you can't repay — no capital at risk
```

```solidity
// Solidity pseudocode — flash swap callback structure
contract FlashArbBot {
    IUniswapV3Pool poolA;

    function executeArb(address token0, address token1, uint256 amount) external {
        // Step 1: initiate flash swap from pool A
        // amount0 = token0 to borrow, amount1 = token1 to borrow
        poolA.flash(address(this), amount, 0, abi.encode(amount));
    }

    // Step 2: Uniswap calls this back after giving us the tokens
    function uniswapV3FlashCallback(
        uint256 fee0, uint256 fee1, bytes calldata data
    ) external {
        uint256 borrowed = abi.decode(data, (uint256));

        // Step 3: Execute arb — sell borrowed token on pool B
        uint256 received = sellOnPoolB(borrowed);

        // Step 4: Verify we profited (safety check)
        uint256 repay = borrowed + fee0;
        require(received > repay, "ARB_UNPROFITABLE");

        // Step 5: Repay flash swap
        IERC20(token0).transfer(address(poolA), repay);

        // Step 6: Profit stays in this contract
        // (surplus = received - repay)
    }
}
```

### Flash Swap Profitability Gate

```python
def flash_swap_viable(borrowed_usd, gross_spread_pct,
                      borrow_fee_pct, eth_price_usd, gas_gwei=30):
    """
    borrow_fee_pct: Uniswap flash fee = pool fee tier (e.g. 0.05% → 0.0005)
    gross_spread_pct: price difference between pool A and pool B
    """
    gross_profit    = borrowed_usd * (gross_spread_pct / 100)
    borrow_fee_cost = borrowed_usd * borrow_fee_pct
    gas_cost        = (GAS_BENCHMARKS['flash_swap_v3_with_arb'] *
                       gas_gwei / 1e9 * eth_price_usd)

    net_profit = gross_profit - borrow_fee_cost - gas_cost

    return {
        "gross_profit_usd":   round(gross_profit, 2),
        "borrow_fee_usd":     round(borrow_fee_cost, 2),
        "gas_cost_usd":       round(gas_cost, 2),
        "net_profit_usd":     round(net_profit, 2),
        "min_spread_to_break": round(
            (borrow_fee_cost + gas_cost) / borrowed_usd * 100, 4
        ),
        "viable":             net_profit > 0,
    }
```

---

## 1E. Single-Side Liquidity Provision

### When Price Is Outside Your Range

If you deposit into a v3 range where the current price is **above** the range,
your entire deposit converts to token0 (the lower-valued token).
If the price is **below** the range, your entire deposit converts to token1.

This is a deliberate strategy — not a bug:

```
SINGLE-SIDE DEPOSIT USE CASES:

  Case 1 — LIMIT ORDER EQUIVALENT
    Place token0 in a range ABOVE current price.
    If price rises into your range, your token0 gradually converts to token1.
    When price passes through your entire range = full conversion = limit sell.

  Case 2 — BULLISH BUY LIMIT
    Place token1 (stablecoin) in a range BELOW current price.
    If price drops into your range, stablecoin converts to token0.
    = Automated buy-the-dip LP position.

  Case 3 — DIRECTIONAL BET WITH FEE INCOME
    Deploy capital in the direction you believe price will move.
    If you are correct, you earn fees as price traverses your range.
    If you are wrong, price never enters your range — zero fees, zero IL.
```

### Single-Side LP Math

```python
def single_side_position(token_deposited, token_is_token0,
                         current_price, p_low, p_high):
    """
    Compute L and position state for a single-token deposit.
    Requires price to be entirely ABOVE range (token0 only)
    or entirely BELOW range (token1 only).
    """
    import math

    if token_is_token0:
        # Deposit token0 only → range must be: p_low < p_high < current_price
        assert p_high < current_price, "Range must be below current price for token0-only deposit"
        spa = math.sqrt(p_low)
        spb = math.sqrt(p_high)
        L = token_deposited / (1/spa - 1/spb)
        return {
            "L": L,
            "token0_deposited": token_deposited,
            "token1_deposited": 0,
            "converts_to_token1_if_price_drops_through_range": True,
            "effective_avg_buy_price": (p_low + p_high) / 2,
        }
    else:
        # Deposit token1 only → range must be: current_price < p_low < p_high
        assert p_low > current_price, "Range must be above current price for token1-only deposit"
        spa = math.sqrt(p_low)
        spb = math.sqrt(p_high)
        L = token_deposited / (spb - spa)
        return {
            "L": L,
            "token0_deposited": 0,
            "token1_deposited": token_deposited,
            "converts_to_token0_if_price_rises_through_range": True,
            "effective_avg_sell_price": (p_low + p_high) / 2,
        }
```

### Single-Side as a Replacement for Limit Orders

```python
def limit_order_via_lp(target_sell_price, slippage_tolerance_pct,
                       current_price, token0_amount, fee_tier):
    """
    Create a narrow single-side range that mimics a limit sell order.
    Narrower = closer to exact price but less fees if slow to fill.
    """
    import math
    # Place a tight range around target price
    width = target_sell_price * (slippage_tolerance_pct / 100)
    p_low  = target_sell_price - width / 2
    p_high = target_sell_price + width / 2

    spacing = {0.0001:1, 0.0005:10, 0.003:60, 0.01:200}[fee_tier]
    t_low  = (math.floor(math.log(p_low)  / math.log(1.0001)) // spacing) * spacing
    t_high = (math.ceil( math.log(p_high) / math.log(1.0001)) // spacing + 1) * spacing

    assert p_low > current_price, "Target price must be above current for a sell limit"

    return {
        "tick_lower":    t_low,
        "tick_upper":    t_high,
        "price_lower":   round(p_low, 4),
        "price_upper":   round(p_high, 4),
        "token0_in":     token0_amount,
        "token1_in":     0,
        "note":          "Remove position when price exits range (order filled)",
    }
```

---

## 1F. Tick Arbitrage & Liquidity Gap Analysis

### How Ticks Create Exploitable Zones

Every initialized tick is a price level where liquidity changes. Between
two consecutive ticks, the AMM behaves like a constant-product curve with
fixed liquidity. When there is a **gap** — a price zone with few or no
initialized ticks — price moves through that zone with very little resistance.

```
Tick density visualization:

  Price:  1800   2000   2200   2400   2600   2800   3000   3200
  Liq:    ████   ████   ██     █      ·      ·      ██     ████
                              ^GAP^   ^GAP^
                         Low resistance zone: price moves fast here
```

```python
def analyse_tick_gaps(ticks, current_tick, lookback_ticks=500):
    """
    ticks: list of { tickIdx, liquidityNet, liquidityGross }
    Returns: list of gap zones with estimated price resistance
    """
    import math
    # Sort ticks
    sorted_ticks = sorted(ticks, key=lambda t: t['tickIdx'])

    # Compute cumulative liquidity at each tick
    cumulative_liq = 0
    liq_by_tick = {}
    for t in sorted_ticks:
        cumulative_liq += int(t['liquidityNet'])
        liq_by_tick[t['tickIdx']] = cumulative_liq

    # Find gaps (consecutive ticks with low liquidity between them)
    gaps = []
    tick_list = sorted(liq_by_tick.keys())
    for i in range(len(tick_list) - 1):
        tick_a    = tick_list[i]
        tick_b    = tick_list[i+1]
        gap_width = tick_b - tick_a
        liq_in_gap = liq_by_tick[tick_a]

        price_a = 1.0001 ** tick_a
        price_b = 1.0001 ** tick_b
        pct_move = (price_b / price_a - 1) * 100

        # Flag as gap if: wide gap AND low liquidity
        if gap_width > 100 and liq_in_gap < 1e18:
            gaps.append({
                "tick_lower":   tick_a,
                "tick_upper":   tick_b,
                "gap_width":    gap_width,
                "price_lower":  round(price_a, 4),
                "price_upper":  round(price_b, 4),
                "pct_move":     round(pct_move, 3),
                "liquidity":    liq_in_gap,
                "above_current": tick_a > current_tick,
                "resistance":   "LOW" if liq_in_gap < 5e17 else "MEDIUM",
            })

    # Sort by gap width descending
    return sorted(gaps, key=lambda g: g['gap_width'], reverse=True)
```

### Tick Arbitrage — Exploiting Price Gaps

A tick arb exploits the fact that a large swap can push price through a
low-liquidity gap much more cheaply than the arber buying the same tokens
on an external CEX. The arber profits from the price impact on the AMM side.

```python
def tick_arb_opportunity(gap, external_price, pool_fee, eth_price_usd,
                         gas_gwei=30):
    """
    gap: a gap zone from analyse_tick_gaps()
    external_price: what the token trades for on Binance/Coinbase
    pool_fee: e.g. 0.003

    If external price is inside the gap → arb exists.
    Buy cheaply from AMM (low resistance) → sell at external price.
    """
    midpoint = (gap['price_lower'] + gap['price_upper']) / 2

    if external_price <= gap['price_lower'] or external_price >= gap['price_upper']:
        return None  # external price not in gap, no arb

    # Estimate price impact of pushing through gap
    # (simplified — full calc requires integrating liquidity curve)
    spread_to_external = abs(external_price - midpoint) / midpoint
    net_spread = spread_to_external - pool_fee

    if net_spread <= 0:
        return None

    # Approximate max arb size = liquidity in gap (very rough)
    approx_usd_arb = gap['liquidity'] / 1e18 * external_price * 0.01

    gas_cost_usd = GAS_BENCHMARKS['uniswap_v3_swap'] * gas_gwei / 1e9 * eth_price_usd

    return {
        "gap":             gap,
        "external_price":  external_price,
        "net_spread_pct":  round(net_spread * 100, 4),
        "approx_profit_usd": round(approx_usd_arb * net_spread, 2),
        "gas_cost_usd":    round(gas_cost_usd, 2),
        "viable":          approx_usd_arb * net_spread > gas_cost_usd,
    }
```

### Using Gaps for LP Range Placement

Beyond arb, gaps tell you where to place LP ranges:

```
STRATEGY: Place LP just ABOVE or BELOW a large gap.

WHY: When price approaches the gap, it will accelerate through it.
     Your position captures fees on the way in AND out.
     Position just inside the gap edge = high fee capture during gap traversal.

RISK: If price shoots through your range quickly, IL accrues fast.
      Use this only in high-fee-tier pools (0.3% or 1%) where fees
      compensate for the rapid traversal.
```

```python
def lp_at_gap_edge(gaps, current_price, fee_tier, capital_usd):
    """
    Place LP just inside the nearest gap edge above and below current price.
    Returns recommended positions (one above, one below).
    """
    above_gaps = [g for g in gaps if g['price_lower'] > current_price]
    below_gaps = [g for g in gaps if g['price_upper'] < current_price]

    recommendations = []

    if above_gaps:
        nearest_above = min(above_gaps, key=lambda g: g['price_lower'])
        recommendations.append({
            "direction":    "ABOVE",
            "tick_lower":   nearest_above['tick_lower'] - 100,
            "tick_upper":   nearest_above['tick_lower'] + 200,
            "price_lower":  nearest_above['price_lower'] * 0.99,
            "price_upper":  nearest_above['price_lower'] * 1.01,
            "capital_usd":  capital_usd * 0.25,   # size small — high risk
            "rationale":    "Capture fees as price enters gap from below",
        })

    if below_gaps:
        nearest_below = max(below_gaps, key=lambda g: g['price_upper'])
        recommendations.append({
            "direction":    "BELOW",
            "tick_lower":   nearest_below['tick_upper'] - 200,
            "tick_upper":   nearest_below['tick_upper'] + 100,
            "price_lower":  nearest_below['price_upper'] * 0.99,
            "price_upper":  nearest_below['price_upper'] * 1.01,
            "capital_usd":  capital_usd * 0.25,
            "rationale":    "Capture fees as price enters gap from above",
        })

    return recommendations
```

---

## 1G. Cross-Chain Swaps

### The Three Layers of Cross-Chain Cost

```
TOTAL CROSS-CHAIN COST =
  Source chain gas cost     (execute on chain A)
+ Bridge fee                (% of amount bridged, typically 0.05–0.30%)
+ Destination chain gas     (execute on chain B)
+ Slippage on source swap   (if swapping before bridge)
+ Slippage on destination   (if swapping after bridge)
+ Time cost                 (opportunity cost of capital locked in bridge, mins–hours)
```

### Bridge Comparison Framework

```python
BRIDGE_PROFILES = {
    "across":     { "fee_pct": 0.06, "time_min": 2,   "safety": "HIGH",   "atomic": False },
    "stargate":   { "fee_pct": 0.06, "time_min": 1,   "safety": "HIGH",   "atomic": False },
    "hop":        { "fee_pct": 0.10, "time_min": 15,  "safety": "HIGH",   "atomic": False },
    "layerzero":  { "fee_pct": 0.05, "time_min": 5,   "safety": "MEDIUM", "atomic": False },
    "ccip":       { "fee_pct": 0.15, "time_min": 10,  "safety": "HIGH",   "atomic": False },
    "wormhole":   { "fee_pct": 0.00, "time_min": 15,  "safety": "MEDIUM", "atomic": False },
}

def cross_chain_cost_model(amount_usd, bridge_name, src_gas_usd, dst_gas_usd,
                            src_slippage_pct=0.05, dst_slippage_pct=0.05):
    bridge = BRIDGE_PROFILES[bridge_name]
    bridge_fee   = amount_usd * bridge['fee_pct'] / 100
    slippage_cost = amount_usd * (src_slippage_pct + dst_slippage_pct) / 100
    total_cost   = bridge_fee + src_gas_usd + dst_gas_usd + slippage_cost
    net_received = amount_usd - total_cost
    total_cost_pct = total_cost / amount_usd * 100

    return {
        "bridge":           bridge_name,
        "amount_usd":       amount_usd,
        "bridge_fee_usd":   round(bridge_fee, 2),
        "gas_cost_usd":     round(src_gas_usd + dst_gas_usd, 2),
        "slippage_cost_usd": round(slippage_cost, 2),
        "total_cost_usd":   round(total_cost, 2),
        "total_cost_pct":   round(total_cost_pct, 3),
        "net_received_usd": round(net_received, 2),
        "time_min":         bridge['time_min'],
        "viable_for_arb":   total_cost_pct < 0.5,  # arb needs very low cost
        "viable_for_rebalance": total_cost_pct < 2.0,
    }
```

### Cross-Chain Arb Viability Rule

Cross-chain arb is almost never viable for retail-sized operations because
bridge fees + dual gas + time risk consume the spread. It is only viable when:

```
1. Spread between chains > 0.5% AND
2. Bridge time < 5 minutes (stale price risk) AND
3. Amount large enough that gas is < 0.1% of amount AND
4. You can SIMULATE destination chain execution before bridging
```

```python
def cross_chain_arb_viable(spread_pct, bridge_profile, amount_usd,
                            src_gas_usd, dst_gas_usd):
    """Capital safety: if bridge hangs, you are stuck mid-trade."""
    profile     = BRIDGE_PROFILES[bridge_profile]
    total_cost  = cross_chain_cost_model(amount_usd, bridge_profile,
                                          src_gas_usd, dst_gas_usd)
    net_spread  = spread_pct - total_cost['total_cost_pct']

    # Safety disqualifiers
    if profile['time_min'] > 5:
        return False, "BRIDGE_TOO_SLOW: price will move during transit"
    if total_cost['total_cost_pct'] > spread_pct * 0.6:
        return False, f"COSTS_EAT_SPREAD: cost={total_cost['total_cost_pct']:.3f}% spread={spread_pct:.3f}%"
    if amount_usd < 50_000:
        return False, "AMOUNT_TOO_SMALL: gas not amortised at this size"

    return net_spread > 0.1, f"NET_SPREAD={net_spread:.3f}%"
```

---

## 1H. Weekly Movement Forecasting

### The Forecasting Hierarchy

```
LAYER 1 — MACRO FILTER (weekly timeframe)
  Is the market in risk-on or risk-off mode?
  Use: BTC 7-day momentum, ETH/BTC ratio, total DeFi TVL trend

LAYER 2 — POOL-LEVEL VOLATILITY FORECAST (daily timeframe)
  What is the expected weekly price range for this specific pool?
  Use: GARCH-inspired vol forecast from subgraph daily data

LAYER 3 — TICK-LEVEL FLOW FORECAST (hours–days)
  Which direction has less resistance in the tick distribution?
  Use: tick gap analysis (Section 1F) + recent swap directional flow

LAYER 4 — POSITION DECISION
  Given layers 1–3, should I: stay, rebalance, single-side, or exit?
```

### Layer 2 — Weekly Volatility Forecast

```python
import numpy as np

def forecast_weekly_vol(daily_closes, ewma_span=10):
    """
    Forecast the expected weekly price range using EWMA volatility.
    Returns: { vol_daily, vol_weekly, expected_high, expected_low,
               range_pct, regime }
    """
    if len(daily_closes) < ewma_span + 1:
        return None

    log_returns = np.log(np.array(daily_closes[1:]) /
                          np.array(daily_closes[:-1]))

    # EWMA variance (decay = 2/(span+1))
    alpha  = 2 / (ewma_span + 1)
    var    = np.var(log_returns[:5])   # seed
    for r in log_returns[5:]:
        var = alpha * r**2 + (1 - alpha) * var

    vol_daily  = var ** 0.5
    vol_weekly = vol_daily * (5 ** 0.5)    # 5 trading days

    current = daily_closes[-1]
    expected_high = current * np.exp(+vol_weekly)
    expected_low  = current * np.exp(-vol_weekly)

    # Regime classification
    if vol_weekly > 0.15:   regime = "HIGH_VOL — widen range or stay flat"
    elif vol_weekly > 0.07: regime = "MEDIUM_VOL — standard range"
    else:                    regime = "LOW_VOL — tighten range for higher APR"

    return {
        "vol_daily_annualised": round(vol_daily * (252**0.5), 4),
        "vol_weekly":           round(vol_weekly, 4),
        "expected_high":        round(expected_high, 4),
        "expected_low":         round(expected_low, 4),
        "range_pct":            round((expected_high / expected_low - 1) * 100, 2),
        "regime":               regime,
    }
```

### Layer 3 — Directional Flow from Swap History

```python
def directional_flow_signal(swaps_last_24h):
    """
    swaps_last_24h: list of { amount0, amount1, timestamp }
    Negative amount0 = bought token0 (bullish)
    Positive amount0 = sold token0 (bearish)
    Returns: { buy_volume, sell_volume, flow_ratio, signal }
    """
    buy_vol  = sum(abs(float(s['amount0'])) for s in swaps_last_24h
                   if float(s['amount0']) < 0)
    sell_vol = sum(abs(float(s['amount0'])) for s in swaps_last_24h
                   if float(s['amount0']) > 0)

    total = buy_vol + sell_vol
    if total == 0:
        return { "signal": "NEUTRAL", "flow_ratio": 0 }

    flow_ratio = (buy_vol - sell_vol) / total   # +1 = all buys, -1 = all sells

    if flow_ratio > 0.20:   signal = "BULLISH — consider single-side above"
    elif flow_ratio < -0.20: signal = "BEARISH — consider single-side below"
    else:                    signal = "NEUTRAL — symmetric range preferred"

    return {
        "buy_volume_usd":  round(buy_vol, 0),
        "sell_volume_usd": round(sell_vol, 0),
        "flow_ratio":      round(flow_ratio, 3),
        "signal":          signal,
    }
```

### Layer 4 — Forecast-Driven Position Decision

```python
def position_decision(vol_forecast, flow_signal, tick_gaps,
                      current_position, config):
    """
    Integrate all three layers into a single position recommendation.
    """
    decisions = []

    # 1. High vol + neutral flow → widen range preemptively
    if "HIGH_VOL" in vol_forecast['regime'] and "NEUTRAL" in flow_signal['signal']:
        decisions.append({
            "action":  "WIDEN_RANGE",
            "reason":  f"High weekly vol {vol_forecast['vol_weekly']:.2%} expected",
            "target_range": [vol_forecast['expected_low'],
                             vol_forecast['expected_high']],
        })

    # 2. Directional flow + nearby tick gap → single-side in direction
    large_gaps_above = [g for g in tick_gaps if g['above_current']
                        and g['pct_move'] > 2 and g['resistance'] == 'LOW']
    if "BULLISH" in flow_signal['signal'] and large_gaps_above:
        decisions.append({
            "action":  "SINGLE_SIDE_ABOVE",
            "reason":  "Bullish flow + low-resistance gap above",
            "gap":     large_gaps_above[0],
            "capital_pct": 0.30,  # deploy 30% of available capital
        })

    # 3. Low vol → tighten range around current price
    if "LOW_VOL" in vol_forecast['regime']:
        decisions.append({
            "action":  "TIGHTEN_RANGE",
            "reason":  "Low vol regime — tight range maximises fee APR",
            "target_range": [
                vol_forecast['expected_low'] * 1.01,
                vol_forecast['expected_high'] * 0.99,
            ],
        })

    # 4. Default: no action
    if not decisions:
        decisions.append({
            "action": "HOLD",
            "reason": "No strong signal. Preserve capital."
        })

    return decisions
```

---

# PHASE 2 — BACKTEST

## Objective

Test the full Workshop 03 toolkit on historical data. Run four parallel
experiments. Every experiment must pass the capital safety gate before
recording a result.

---

## Experiment A — Two-Pool Arb Frequency & Viability

```python
def backtest_two_pool_arb(pool_a_daily, pool_b_daily, eth_prices,
                           gas_gwei_series, fee_a=0.0005, fee_b=0.003):
    """
    For each day: check if spread > fees + gas. Count viable opportunities.
    pool_a_daily, pool_b_daily: list of { date, close }
    """
    results = []
    for i, (da, db) in enumerate(zip(pool_a_daily, pool_b_daily)):
        arb = find_two_pool_arb(
            da['close'], db['close'], fee_a, fee_b,
            10_000, eth_prices[i], gas_gwei_series[i]
        )
        if arb and arb['viable']:
            results.append({
                "date":            da['date'],
                "spread_pct":      arb['spread_raw_pct'],
                "net_profit_usd":  arb['net_profit_usd'],
                "gas_cost_usd":    arb['gas_cost_usd'],
            })

    viable_days = len(results)
    total_days  = len(pool_a_daily)
    return {
        "viable_arb_days":    viable_days,
        "total_days":         total_days,
        "frequency_pct":      round(viable_days / total_days * 100, 1),
        "avg_net_profit_usd": round(sum(r['net_profit_usd'] for r in results)
                                    / max(viable_days, 1), 2),
        "max_profit_day_usd": round(max((r['net_profit_usd'] for r in results), default=0), 2),
        "results":            results,
    }
```

---

## Experiment B — Single-Side LP vs Standard LP

Compare a **single-side directional LP** (using the flow signal) against
a standard symmetric range over 30-day forward windows.

```python
def backtest_single_side_vs_symmetric(pool_history, flow_signals,
                                       initial_capital=10_000):
    """
    flow_signals: daily { date, signal: BULLISH/BEARISH/NEUTRAL }
    On each signal day: open single-side in signal direction (if BULLISH/BEARISH)
    vs opening symmetric range.
    """
    single_results, symm_results = [], []

    for i, day in enumerate(pool_history[:-30]):
        signal = flow_signals[i]['signal']
        if 'NEUTRAL' in signal:
            continue   # only trade on directional signals

        future = pool_history[i:i+30]
        price_entry = day['close']
        price_30d   = future[-1]['close']
        price_move  = (price_30d / price_entry) - 1

        # Single-side: capture if price moves in signal direction
        if 'BULLISH' in signal:
            vol = daily_vol(pool_history, i, 14)
            p_low  = price_entry * 1.01   # range starts just above current
            p_high = price_entry * (1 + vol * (5**0.5))
            in_range = p_low <= price_30d <= p_high
            fee_income = sum(d['feesUSD'] for d in future
                             if p_low <= d['close'] <= p_high) * \
                         (initial_capital / day['tvlUSD'])
            single_results.append({
                "in_range": in_range, "price_move": price_move,
                "fee_income": fee_income, "capital": initial_capital,
                "net": fee_income - (0 if in_range else 0),  # no IL if never entered
            })

        # Symmetric: standard ±1-sigma range
        vol = daily_vol(pool_history, i, 14)
        p_low_s  = price_entry * (1 - vol * (5**0.5))
        p_high_s = price_entry * (1 + vol * (5**0.5))
        fee_s = sum(d['feesUSD'] for d in future
                    if p_low_s <= d['close'] <= p_high_s) * \
                (initial_capital / day['tvlUSD'])
        il_s  = il_full_range(price_30d / price_entry) * initial_capital
        symm_results.append({
            "fee_income": fee_s, "il": il_s,
            "net": fee_s + il_s
        })

    return {
        "single_side_avg_net": round(np.mean([r['net'] for r in single_results]), 2),
        "symmetric_avg_net":   round(np.mean([r['net'] for r in symm_results]), 2),
        "single_side_in_range_pct": round(
            sum(1 for r in single_results if r['in_range']) / len(single_results) * 100, 1
        ),
        "winner": "SINGLE_SIDE" if np.mean([r['net'] for r in single_results]) >
                                   np.mean([r['net'] for r in symm_results]) else "SYMMETRIC",
    }
```

---

## Experiment C — Tick Gap Strategy

Test whether placing LP at gap edges outperforms random range placement
on the same pool.

```python
def backtest_gap_edge_lp(pool_history, tick_history, initial_capital=10_000):
    """
    tick_history: { date → list of ticks } fetched at each rebalance point
    """
    gap_results, random_results = [], []

    for i in range(0, len(pool_history)-30, 30):
        day   = pool_history[i]
        future = pool_history[i:i+30]
        price  = day['close']
        ticks  = tick_history.get(day['date'], [])
        if not ticks: continue

        # Gap edge placement
        gaps = analyse_tick_gaps(ticks, int(math.log(price) / math.log(1.0001)))
        above = [g for g in gaps if g['price_lower'] > price and g['pct_move'] > 1]
        if not above: continue
        gap_range = (above[0]['price_lower'] * 0.99, above[0]['price_lower'] * 1.01)

        # Standard vol-based placement
        vol = daily_vol(pool_history, i, 14)
        std_range = (price * (1 - vol), price * (1 + vol))

        for (p_l, p_h), results in [(gap_range, gap_results),
                                     (std_range, random_results)]:
            fees = sum(d['feesUSD'] for d in future
                       if p_l <= d['close'] <= p_h) * \
                   (initial_capital / day['tvlUSD'])
            results.append({ "fee_income": fees,
                              "in_range_days": sum(1 for d in future
                                                   if p_l <= d['close'] <= p_h) })

    return {
        "gap_avg_fee_income": round(np.mean([r['fee_income'] for r in gap_results]), 2),
        "std_avg_fee_income": round(np.mean([r['fee_income'] for r in random_results]), 2),
        "gap_avg_in_range_days": round(np.mean([r['in_range_days'] for r in gap_results]), 1),
        "std_avg_in_range_days": round(np.mean([r['in_range_days'] for r in random_results]), 1),
    }
```

---

## Backtest Output Format

```json
{
  "workshop": "03",
  "phase": "backtest",
  "period": "2025-06-01 to 2026-02-01",

  "experiment_A_two_pool_arb": {
    "pool_pair":        "WETH/USDC 0.05% vs 0.3%",
    "viable_arb_days":  14,
    "total_days":       244,
    "frequency_pct":    5.7,
    "avg_net_profit_usd": 23.40,
    "finding":          "Arb only viable on high-vol days when spread > 0.08%"
  },

  "experiment_B_single_vs_symmetric": {
    "pool":             "WETH/USDC 0.05%",
    "single_side_avg_net_usd": 87.20,
    "symmetric_avg_net_usd":   54.30,
    "single_side_in_range_pct": 61.4,
    "winner":           "SINGLE_SIDE",
    "finding":          "Flow signal adds 60% to net income when directional signal fires"
  },

  "experiment_C_gap_edge_lp": {
    "pool":             "WETH/USDC 0.05%",
    "gap_avg_fee_income_usd":  142.10,
    "std_avg_fee_income_usd":   98.40,
    "gap_avg_in_range_days":    8.2,
    "std_avg_in_range_days":   12.6,
    "finding":          "Gap edge earns more fees but spends fewer days in range. High fee tier essential."
  },

  "capital_safety_events": {
    "gate_rejections":   31,
    "rejection_reasons": { "GAS_EXCEEDS_PROFIT": 18, "BELOW_MIN_PROFIT": 9, "SIZE_EXCEEDS_LIMIT": 4 },
    "capital_preserved_usd": 710
  }
}
```

---

# PHASE 3 — EXECUTE

## The Workshop 03 Daily Ops Sequence

```
EVERY DAY:
  1. Fetch current prices (pool + external CEX)
  2. Run two-pool arb scanner across tracked pool pairs
  3. Analyse tick distribution → gap map
  4. Run weekly vol forecast (EWMA)
  5. Run directional flow signal (last 24h swaps)
  6. Integrate → position decision
  7. For each flagged action → gas gate → safety gate → execute
  8. Log everything
```

## Execution Decision Tree

```
                    START
                      │
         ┌────────────┴───────────────┐
         ▼                            ▼
   ARB VIABLE?                  FLOW SIGNAL?
   (spread > cost)               (BULLISH/BEARISH)
         │                            │
   YES → flash swap or         YES → check nearest gap
         two-pool route               │
         │                    gap + signal aligned?
         │                            │
         │                    YES → single-side LP
         │                    NO  → symmetric or HOLD
         │
   ┌─────┴────────────────────────────┐
   ▼                                  ▼
GAS GATE                        SAFETY GATE
estimate_gas_cost_usd()         OperationSafety.gate()
viable? → proceed               approved? → proceed
no? → log and skip              no? → HOLD
```

## Live Operation Log Format

```json
{
  "workshop":  "03",
  "phase":     "execute",
  "timestamp": "2026-02-17T10:15:00Z",

  "market_state": {
    "eth_price_usd":     3210.45,
    "base_fee_gwei":     22.4,
    "vol_weekly_forecast": 0.089,
    "regime":           "MEDIUM_VOL",
    "flow_signal":      "BULLISH",
    "flow_ratio":       0.31
  },

  "tick_analysis": {
    "gaps_above": [
      { "price_lower": 3280, "price_upper": 3340, "pct_move": 1.83, "resistance": "LOW" }
    ],
    "gaps_below": [
      { "price_lower": 3100, "price_upper": 3140, "pct_move": 1.27, "resistance": "MEDIUM" }
    ]
  },

  "arb_scan": {
    "pair":           "WETH/USDC 0.05% vs 0.3%",
    "spread_pct":     0.04,
    "viable":         false,
    "reason":         "spread below gas breakeven"
  },

  "decision": {
    "action":         "SINGLE_SIDE_ABOVE",
    "capital_usd":    2000,
    "range":          [3280, 3340],
    "rationale":      "BULLISH flow + LOW resistance gap directly above"
  },

  "safety_gate": {
    "approved":       true,
    "gas_estimate_usd": 4.20,
    "net_after_gas": "positive if traversed"
  },

  "execution": {
    "tx_hash":        "0x...",
    "position_id":    "67890",
    "gas_used_usd":   4.18,
    "status":         "CONFIRMED"
  }
}
```

---

# PHASE 4 — COLLECT RESULTS

## Weekly Audit Template

```python
def weekly_audit(ops_log_path, portfolio_value_start):
    """Summarise all Workshop 03 operations over the past 7 days."""
    ops = load_json(ops_log_path)

    arb_ops       = [o for o in ops if o['decision']['action'] in
                     ['TWO_POOL_ARB', 'FLASH_SWAP']]
    single_side   = [o for o in ops if 'SINGLE_SIDE' in o['decision']['action']]
    gap_lp        = [o for o in ops if o['decision']['action'] == 'GAP_EDGE_LP']
    held          = [o for o in ops if o['decision']['action'] == 'HOLD']
    rejected      = [o for o in ops if not o['safety_gate']['approved']]

    total_gas     = sum(o['execution']['gas_used_usd'] for o in ops
                        if 'execution' in o)
    total_profit  = sum(o.get('realised_profit_usd', 0) for o in ops)

    return {
        "week":                  "YYYY-WW",
        "ops_attempted":         len(ops),
        "ops_rejected_by_gate":  len(rejected),
        "capital_saved_by_gate": sum(o['decision']['capital_usd']
                                     for o in rejected),
        "arb_ops":               len(arb_ops),
        "single_side_ops":       len(single_side),
        "gap_edge_lp_ops":       len(gap_lp),
        "hold_decisions":        len(held),
        "total_gas_cost_usd":    round(total_gas, 2),
        "total_realised_profit": round(total_profit, 2),
        "net_profit":            round(total_profit - total_gas, 2),
        "return_on_portfolio":   round((total_profit - total_gas) /
                                       portfolio_value_start * 100, 3),
        "safety_gate_save_rate": round(len(rejected) / len(ops) * 100, 1),
    }
```

## Results Format

```json
{
  "workshop":  "03",
  "phase":     "collect",
  "week":      "2026-W08",

  "operations_summary": {
    "total":              47,
    "rejected_by_gate":   31,
    "capital_preserved":  7140,
    "executed":           16
  },

  "strategy_breakdown": {
    "two_pool_arb":    { "count": 3, "net_profit": 41.20, "avg_gas": 8.40 },
    "flash_swap":      { "count": 1, "net_profit": 112.30, "avg_gas": 14.10 },
    "single_side_lp":  { "count": 8, "net_profit": 218.40, "avg_gas": 4.20 },
    "gap_edge_lp":     { "count": 4, "net_profit": 94.10,  "avg_gas": 4.20 }
  },

  "portfolio_result": {
    "total_gross_profit_usd": 465.00,
    "total_gas_cost_usd":      72.40,
    "net_profit_usd":          393.00,
    "return_pct":              1.31,
    "annualised_pct":          68.1
  },

  "capital_safety_summary": {
    "max_drawdown_usd":      0,
    "reverted_txs":          0,
    "largest_single_loss":   0,
    "gate_effectiveness":    "31 of 47 ops blocked, $7,140 capital preserved"
  },

  "model_updates": {
    "min_spread_arb":        { "old": "0.04%", "new": "0.06%", "reason": "gas higher than modelled" },
    "flow_signal_threshold": { "old": 0.20,    "new": 0.25,    "reason": "weak signals had low IC" },
    "gap_min_pct_move":      { "old": 1.0,     "new": 1.5,     "reason": "small gaps reverted fast" }
  }
}
```

---

## Closing the Loop

### What to Write in STATE.md

```markdown
## WORKSHOP_03 — Completed [DATE]

### Safety Gate Performance
- Total ops evaluated: [n]
- Gate rejection rate: [pct]%
- Capital preserved by gate: $[amount]
- Reverted transactions: [n] — target is 0

### Best Strategy This Week
[ARB / FLASH / SINGLE_SIDE / GAP_LP]: net $[amount], [n] ops

### Model Parameter Updates
- Min arb spread: [old] → [new]
- Flow signal threshold: [old] → [new]
- Gap min pct move: [old] → [new]
- Gas safety multiplier: [old] → [new]

### Forecasting Accuracy
- Directional signal correct: [pct]%
- Vol forecast within 20% of actual: [pct]%

### WORKSHOP_03: SCHEMA_NOTES
- [tick query surprises]
- [swap event fields used]

### Capital Safety Status
Portfolio intact: YES / NO
If NO — stop all ops, diagnose before Workshop 04.
```

---

## The Non-Negotiable Exit Condition

```python
WORKSHOP_03_STOP_CONDITIONS = [
    "Any single operation results in > 5% portfolio loss",
    "Two consecutive reverted transactions on the same op type",
    "Gas costs exceed 30% of gross profits for the week",
    "Portfolio drawdown exceeds 10% from workshop start value",
    "Bridge transaction unconfirmed for > 30 minutes",
]
# If ANY condition is met: STOP. Update STATE.md. Do not continue.
# Diagnose the failure. Fix the model. Run backtests again. Then resume.
```

---

## Workshop Progression

```
Workshop 01 ✓  Signal scoring, pool ranking, LP allocation
Workshop 02 ✓  Deposits, fees, rebalancing, IL, ONL, net P&L
Workshop 03 ✓  Arb, flash swaps, tick gaps, single-side, forecasting

        │
        ▼
Workshop 04
  Multi-position portfolio management:
  correlation between positions, delta-neutral construction,
  JIT liquidity detection, sandwich awareness,
  automated monitoring with on-chain event triggers,
  gas batching to reduce operational costs
```

---

## Files to Commit After This Workshop

```
evals/backtest_results_workshop03.json    ← A/B/C experiment results
evals/ops_log_workshop03_YYYYMMDD.json    ← daily operation log
evals/weekly_audit_workshop03.json        ← weekly P&L summary
STATE.md                                  ← WORKSHOP_03 entries
```

---

*UniClaw Workshop 03 | https://github.com/ylintern/uniclawskills | MIT License*
