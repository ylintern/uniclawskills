# UniClaw Skills Repository

This repository now contains the processed outputs of two skill archives:


## Skill Product Overview

UniClaw is a professional Uniswap LP quant skill product with:

- Core AMM knowledge base (`SKILL.md`)
- Identity and operating ethos (`IDENTITY.md`, `SOUL.md`)
- Session state template (`STATE.md`)
- Role-based sub-agent skills (`skills/*.md`)
- Supporting documentation (`docs/`)
- Evaluation scenarios (`evals/evals.json`)

## Repository Structure

```text
.
├── SKILL.md
├── IDENTITY.md
├── SOUL.md
├── STATE.md
├── skills/
│   ├── lp-manager.md
│   ├── strategist.md
│   ├── backtester.md
│   ├── swap-arb.md
│   └── sentiment-analyst.md
├── docs/
│   ├── CONTRIBUTING.md
│   └── RELEASE_SUMMARY.md
├── evals/
│   └── evals.json
├── multiagentskill.zip
└── uniclawskillfile.zip
```

## What Was Processed

1. Extracted and integrated multi-agent skill files from `multiagentskill.zip`.
2. Added a dedicated `SOUL.md` to separate ethos from operating identity.
3. Preserved and kept available source archives for traceability.
4. Kept release/evaluation docs grouped under `docs/` and `evals/`.

## How to Use

1. Start with `STATE.md` for current session context.
2. Read `IDENTITY.md` + `SOUL.md` for behavior and constraints.
3. Use `SKILL.md` for core quantitative LP knowledge.
4. Delegate role-specific tasks via files under `skills/`.
5. Run checks using prompts in `evals/evals.json`.
