# UniClaw Skills Repository

UniClaw is a structured skill repository for Uniswap LP research, strategy design, and multi-agent execution workflows.

## Skill Product Overview

This repository includes:

- Core AMM/LP knowledge base (`SKILL.md`)
- Identity and operating ethos (`IDENTITY.md`, `SOUL.md`)
- Session state template (`STATE.md`)
- Role-based specialist agents (`skills/*.md`)
- Maintainer documentation (`docs/`)
- Workshop playbooks (`docs/workshops/`)
- Evaluation scenarios (`evals/evals.json`)
- Visual asset (`assets/cover.jpg`)

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
│   ├── RELEASE_SUMMARY.md
│   └── workshops/
│       ├── WORKSHOP_01.md
│       └── WORKSHOP_02.md
├── evals/
│   └── evals.json
└── assets/
    └── cover.jpg
```

## Repository Cleanup and Refactor

- Removed obsolete uploaded archive files (`multiagentskill.zip`, `uniclawskillfile.zip`).
- Reorganized workshop material under `docs/workshops/`.
- Moved loose image content into `assets/`.
- Updated this README to match the current canonical layout.

## How to Use

1. Start with `STATE.md` for current session context.
2. Read `IDENTITY.md` + `SOUL.md` for behavior and constraints.
3. Use `SKILL.md` for core quantitative LP knowledge.
4. Delegate role-specific tasks via files under `skills/`.
5. Run checks using prompts in `evals/evals.json`.
