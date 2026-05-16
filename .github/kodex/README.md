# Kodex — Theia Knowledge Base & AI Development Toolkit

> **Purpose:** Day-0 onboarding resource for engineers building a custom IDE on top of Eclipse Theia.
> All files in this directory are **product-side** artifacts — they describe how to *extend* Theia without modifying its core.

## License Reminder

This repository is licensed under **EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0**.
- Any file you add inside this repo must include the SPDX header.
- Extensions you ship as separate npm packages can use any license — EPL does not propagate to plugin consumers.
- Never copy Theia source verbatim into a closed-source package without legal review.

---

## Directory Layout

```
.github/kodex/
├── README.md                        ← you are here
├── knowledge/
│   ├── 00-architecture.md           ← full system breakdown
│   ├── 01-extension-development.md  ← how to build on top of Theia
│   ├── 02-ai-system.md              ← AI subsystem (agents, LLMs, tools)
│   ├── 03-commands-cheatsheet.md    ← build / test / run commands
│   ├── 04-epl-guide.md             ← license compliance for products
│   ├── 05-ai-workflow.md            ← AI-assisted development workflow
│   └── 06-look-and-feel.md         ← theming, colors, fonts, styling
├── agents/
│   ├── theia-planner.agent.md       ← grill mode: planning before any creation ← START HERE
│   ├── theia-architect.agent.md     ← architecture decisions (after planning)
│   ├── theia-coder.agent.md         ← implementing Theia extensions
│   └── theia-reviewer.agent.md      ← code review for Theia PRs
├── instructions/
│   ├── planning-grill.instructions.md     ← mandatory planning gate (applies to all)
│   ├── extension-coding.instructions.md   ← rules for writing Theia extensions
│   ├── ai-agent-coding.instructions.md    ← rules for writing custom AI agents
│   └── testing.instructions.md            ← testing conventions
└── skills/
    ├── feature-planning.md             ← planning workflow & FDD template ← START HERE
    ├── create-theia-extension.md       ← scaffold a new Theia package (requires FDD)
    ├── create-ai-agent.md              ← scaffold a new AI agent (requires FDD)
    └── customize-look-and-feel.md      ← brand colors, themes, CSS variables (requires FDD)
```

## Quick-Start for New Engineers

1. Read [`knowledge/00-architecture.md`](knowledge/00-architecture.md) first — monorepo layout, DI, extension points.
2. Read [`knowledge/01-extension-development.md`](knowledge/01-extension-development.md) — the *how* of building features.
3. Read [`knowledge/02-ai-system.md`](knowledge/02-ai-system.md) — how AI agents and LLMs plug in.
4. **Before building anything new:** run `skills/feature-planning.md` with `agents/theia-planner.agent.md` loaded — this is mandatory.
5. Use the creation **skills** only after a confirmed Feature Design Document (FDD).
6. Use `agents/theia-coder.agent.md` + `instructions/extension-coding.instructions.md` when writing code.

## How to Use AI Assistance in This Project

| Task | Load this agent/instruction |
|---|---|
| **Any new feature — START HERE** | `agents/theia-planner.agent.md` + `skills/feature-planning.md` |
| Architecture decisions (after FDD) | `agents/theia-architect.agent.md` |
| Writing extension code | `agents/theia-coder.agent.md` + `instructions/extension-coding.instructions.md` |
| Reviewing a PR | `agents/theia-reviewer.agent.md` |
| Scaffolding a new package (after FDD) | `skills/create-theia-extension.md` |
| Adding a custom AI agent (after FDD) | `skills/create-ai-agent.md` + `instructions/ai-agent-coding.instructions.md` |
| Changing look & feel (after FDD) | `skills/customize-look-and-feel.md` + `knowledge/06-look-and-feel.md` |
