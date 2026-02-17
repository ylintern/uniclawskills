---
name: swap-arb
version: 1.0.0
role: Swap routing, token ratio management and arbitrage detection
requires: SKILL.md
---

# Swap & Arb — Role Skill

Handles token ratio management after fee collection and detects
arbitrage opportunities between pools. Supports LP rebalancing.

---

## Role Objective

Answer two questions:
1. **After collecting fees, how do I swap back into the right ratio for my position?**
2. **Is there a price discrepancy between pools worth capturing?**

---

## Token Ratio Management (post-fee collection)

After collecting fees, tokens may not be in the right ratio to add back as liquidity.

```
WORKFLOW
────────
1. Know current position range (tickLower, tickUpper) and current tick
2. Calculate target ratio:
     if currentTick < tickLower → 100% token0 needed
     if currentTick > tickUpper → 100% token1 needed
     if in range → calculate split using liquidity math

3. Quote swap via Gateway:
     GET /connectors/uniswap/clmm/quote-swap
     params: tokenIn, tokenOut, amount, fee

4. Check: swap output > gas cost? → proceed
5. Execute via Gateway:
     POST /connectors/uniswap/clmm/execute-swap

6. Return new balances to master for add-liquidity call
```

## Ratio Calculation

```python
def calculate_needed_ratio(current_tick, tick_lower, tick_upper, amount_usd):
    """
    Returns: (amount0, amount1) in token units
    """
    sqrt_lower = tick_to_sqrt_price(tick_lower)
    sqrt_upper = tick_to_sqrt_price(tick_upper)
    sqrt_current = tick_to_sqrt_price(current_tick)

    if current_tick < tick_lower:
        # All in token0
        return (amount_usd / price, 0)
    elif current_tick >= tick_upper:
        # All in token1
        return (0, amount_usd)
    else:
        # Split — use liquidity math
        pct_token0 = (sqrt_upper - sqrt_current) / (sqrt_upper - sqrt_lower)
        pct_token1 = 1 - pct_token0
        return (amount_usd * pct_token0 / price, amount_usd * pct_token1)
```

## Arbitrage Detection

```
SCAN WORKFLOW
─────────────
1. Get price for pair on pool A (fee tier X)
2. Get price for same pair on pool B (fee tier Y)
3. Price difference = abs(priceA - priceB) / priceA

4. Profitable if:
     profit = difference * trade_size
     profit > gas_cost + slippage_cost

5. If profitable: report to UniClaw master with:
     - Entry pool, exit pool
     - Amount, expected profit
     - Gas estimate
     - Risk (slippage, front-run)

6. NEVER execute arb without master confirmation
```

## What to Return to UniClaw

```
SWAP REPORT
───────────
Action:     [Ratio rebalance | Arb opportunity]
From:       [N token0 + N token1]
To:         [N token0 + N token1]
Swap:       [sell N tokenX for N tokenY]
Gas:        $[N]
Net gain:   $[N]
Slippage:   [N]%

Proceed?    [Awaiting master approval]
```

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial role definition | — | — |
