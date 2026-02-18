# UniClaw Skills Repository

UniClaw is a practical skill system for **Uniswap-focused research, strategy design, and multi-agent execution workflows**.

It is designed as a composable operating layer for quantitative LP work: each document defines either core domain knowledge, execution identity, specialist role behavior, or evaluation logic.

---

## What this repository contains

- **Core protocol and LP knowledge** in `SKILL.md`
- **Agent behavior model** in `IDENTITY.md` and `SOUL.md`
- **Session memory scaffold** in `STATE.md`
- **Specialist operator roles** under `skills/`
- **Contributor and release docs** under `docs/`
- **Workshop material** in both `docs/workshops/` and root workshop guides
- **Evaluation scenarios** in `evals/evals.json`
- **Visual branding asset** in `assets/cover.jpg`

---

## Repository map

```text
.
├── SKILL.md
├── IDENTITY.md
├── SOUL.md
├── STATE.md
├── README.md
├── LICENSE
├── WORKSHOP_03.md
├── WORKSHOP_04.md
├── workshop5.md
├── skills/
│   ├── lp-manager.md
│   ├── strategist.md
│   ├── backtester.md
│   ├── swap-arb.md
│   └── sentiment-analyst.md
├── docs/
│   ├── CONTRIBUTING.md
│   ├── RELEASE_SUMMARY.md
│   └── workshops/
│       ├── WORKSHOP_01.md
│       └── WORKSHOP_02.md
├── evals/
│   └── evals.json
└── assets/
    └── cover.jpg
```

---

## Quick start

1. **Set context first**: read `STATE.md` to understand current operating assumptions.
2. **Load identity constraints**: read `IDENTITY.md` and `SOUL.md` before planning or execution.
3. **Ground in domain logic**: use `SKILL.md` as the primary AMM/LP reference.
4. **Select specialists**: route tasks to the appropriate file in `skills/`.
5. **Validate output quality**: run scenario prompts from `evals/evals.json`.

---

## Role guides in `skills/`

Use these files as role-specific prompt overlays for multi-agent decomposition:

- `lp-manager.md` — LP range/risk operations
- `strategist.md` — strategy design and portfolio logic
- `backtester.md` — historical simulation workflows
- `swap-arb.md` — swap and arbitrage opportunity analysis
- `sentiment-analyst.md` — market narrative and sentiment interpretation

---

## Workshops and enablement docs

- `docs/workshops/WORKSHOP_01.md`
- `docs/workshops/WORKSHOP_02.md`
- `WORKSHOP_03.md`
- `WORKSHOP_04.md`
- `workshop5.md`

These are practical walkthroughs for onboarding, simulation drills, and operating cadence.

---

## Contribution and maintenance

- See `docs/CONTRIBUTING.md` for contribution expectations.
- See `docs/RELEASE_SUMMARY.md` for release-level change tracking.

When updating repository structure, keep this README synchronized with actual paths to avoid drift.

---

## Design intent

UniClaw separates concerns deliberately:

- **Knowledge** (`SKILL.md`) should evolve independently from
- **Behavioral constraints** (`IDENTITY.md`, `SOUL.md`) and
- **Execution roles** (`skills/*.md`).

This lets teams upgrade strategy logic without rewriting operating identity, while still preserving repeatable, auditable decision workflows.
