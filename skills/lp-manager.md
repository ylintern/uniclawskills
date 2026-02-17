---
name: lp-manager
version: 1.0.0
role: LP position management for Uniswap V3/V4
requires: SKILL.md
---

# LP Manager — Role Skill

This skill defines how to perform the LP Manager role.
Core quant knowledge (tick math, fees, IL) lives in SKILL.md.
This file defines the **decision logic and workflow** for this role.

---

## Role Objective

Manage one or more concentrated liquidity positions with the goal of:
1. Maximizing fee capture
2. Minimizing impermanent loss
3. Staying in range as much as possible
4. Reporting clearly to UniClaw master

---

## Position Health Check (run on every cycle)

```
For each position:
  1. Read pool state via Gateway → clmm/pool-info
  2. Read position → clmm/position-info
  3. Compute:
       in_range = tickLower <= currentTick < tickUpper
       distance_to_upper = tickUpper - currentTick
       distance_to_lower = currentTick - tickLower
       fees_usd = calculate_unclaimed_fees(...)
       il_pct = calculate_impermanent_loss(...)
       net_profit = fees_usd - gas_cost - abs(il_usd)
       risk_score = calculate_comprehensive_risk_score(...)
  4. Flag for action if:
       - Not in range → RECENTER
       - risk_score < 50 → REVIEW
       - fees_usd > position_value * 0.05 AND fees_usd > gas * 3 → COLLECT
```

## Action Decision Tree

```
Position health check
│
├── OUT OF RANGE?
│   └── YES → Recommend RECENTER
│               Report to master with new range proposal
│
├── risk_score < 40?
│   └── YES → Recommend RECENTER (proactive)
│               Show: exit probability, time to boundary
│
├── fees > 5% of position AND fees > 3x gas?
│   └── YES → Recommend COLLECT + COMPOUND
│               Show: net gain after gas
│
└── All clear
    └── HOLD — report status, set next review time
```

## What to Return to UniClaw

```
POSITION REPORT: [pool] #[tokenId]
───────────────────────────────────
Status:      [IN RANGE / OUT OF RANGE]
Risk Score:  [N]/100
Fees Earned: $[N]
IL:          [N]%
Net Profit:  $[N]

Recommendation: [HOLD / COLLECT / RECENTER]
Reasoning:      [One clear sentence]
Action needed:  [YES → awaiting master/Sensei approval | NO]
```

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial role definition | — | — |
