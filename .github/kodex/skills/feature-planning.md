# Skill: Feature Planning (Grill Mode)

**When to use:** Before creating ANYTHING new — a package, extension, command, widget, AI agent, theme, or any other feature. This skill is the mandatory first step.

**When NOT to use:** You are already past planning and have a completed, user-confirmed Feature Design Document (FDD). In that case, go directly to the relevant creation skill.

---

## Overview

This skill walks you through a structured planning session using the knowledge base and a live codebase search to validate every assumption before a single file is created.

**Output:** A Feature Design Document (FDD) confirmed by the user.

**Gate:** No creation skill (`create-theia-extension.md`, `create-ai-agent.md`, `customize-look-and-feel.md`) should be invoked until this skill produces a confirmed FDD.

---

## Phase 0 — Set Expectations

Before asking any questions, set the context:

> "Before we write anything, I want to run through a short planning session. It'll take a few minutes and saves us from building the wrong thing. I'll ask questions in groups — answer what you know; flag what's still unclear."

---

## Phase 1 — Core Questions

Ask these for every feature, no matter how small.

| # | Question | Why it matters |
|---|---|---|
| 1 | What problem does this solve? | Prevents "solution looking for a problem" |
| 2 | Who is the user? (developer / end user / background service) | Determines UI vs headless, UX needs |
| 3 | What does done look like? (2–3 acceptance criteria) | Gives implementation a finish line |
| 4 | New package or addition to an existing one? | Determines scaffold vs extension |
| 5 | Is this blocking something or exploratory? | Helps prioritize planning depth |

**Done criteria for Phase 1:** All 5 questions have concrete answers. "I don't know yet" on Q5 is fine; "I don't know yet" on Q1–Q3 means we are not ready.

---

## Phase 2 — Technical Shape

Select the relevant sub-group based on Phase 1 answers.

### 2A — General Extension / Feature

| # | Question |
|---|---|
| 6 | Which Theia extension point fits? Use the table below as a reference. |
| 7 | Frontend only, backend only, or both? |
| 8 | Does it need to persist state? (preferences, workspace storage, user storage) |
| 9 | Does it need frontend ↔ backend RPC? |
| 10 | What `@theia/*` packages are needed as dependencies? |

**Extension Point Quick Reference** (from `knowledge/01-extension-development.md` and `.prompts/project-info.prompttemplate`):

| What you want to add | Extension point | Package |
|---|---|---|
| A command (palette action) | `CommandContribution` | `@theia/core` |
| A menu item | `MenuContribution` | `@theia/core` |
| A keyboard shortcut | `KeybindingContribution` | `@theia/core` |
| A new panel / view / tree | `AbstractViewContribution` | `@theia/core` |
| A toolbar item on a tab bar | `TabBarToolbarContribution` | `@theia/core` |
| A main application toolbar item | `ToolbarContribution` | `@theia/core` |
| A status bar item | `StatusBarContribution` | `@theia/core` |
| App startup hook | `FrontendApplicationContribution` | `@theia/core` |
| A preference setting | `PreferenceContribution` | `@theia/core` |
| A language feature (hover, completion) | `LanguageClientContribution` | `@theia/languages` |
| A debug adapter | `DebugAdapterContribution` | `@theia/debug` |
| A task type | `TaskProviderContribution` | `@theia/task` |

### 2B — AI Feature

| # | Question |
|---|---|
| 11 | New agent or extension of an existing one? If extending, which one? |
| 12 | What tools does it need? (list existing tools in `ai-core` first) |
| 13 | What context variables are injected into the prompt? (files, selection, project info) |
| 14 | LLM requirements? (default model vs. specific model ID) |
| 15 | Does it need a custom frontend UI (chat widget variant, tool call display)? |

**Built-in AI tools reference** (from `knowledge/02-ai-system.md`):
- `getFileContent`, `writeFile`, `listDirectory` — filesystem
- `runTerminalCommand` — terminal
- `searchInWorkspace` — file search
- `getWorkspaceFileList` — project structure

### 2C — UI / Look & Feel

| # | Question |
|---|---|
| 16 | Application-wide branding, single component, or full new theme? |
| 17 | New semantic color token, or override of existing `--theia-*` variable? |
| 18 | Are dark, light, and high-contrast variants all defined? |
| 19 | WCAG 2.2 AA contrast ratios verified for all variants? (4.5:1 text, 3:1 UI) |

**CSS Variable Naming Rule:** `editor.background` → `--theia-editor-background` (dots to dashes, `theia-` prefix)

---

## Phase 3 — Codebase Validation

Do not skip this phase. Use search to confirm answers.

### Checklist

- [ ] **Searched `packages/`** for any existing feature that is similar or equivalent
  - Use semantic search: "feature name" + "what it does"
  - Use grep: relevant contribution interface names
- [ ] **Confirmed extension point is correct** — cross-reference `knowledge/01-extension-development.md`
- [ ] **AI feature only:** Checked `packages/ai-core/src/common/` for existing tools/variables
- [ ] **UI feature only:** Checked `packages/core/src/browser/style/` for existing tokens

If a similar feature is found, surface it explicitly:

> "I found `packages/<name>/` which already provides <X>. Does your feature differ in <Y>, or should we extend that?"

---

## Phase 4 — Risk Sweep

Always ask these before producing the FDD.

| # | Question |
|---|---|
| R1 | Does this require modifying any file inside an existing `@theia/*` package? If yes, why can't an extension point be used? |
| R2 | Is any third-party code being copied in? What is its license? |
| R3 | Does this add, remove, or change any exported public API? Breaking or additive? |
| R4 | If R3 is yes — is the new API tagged `@experimental` (safe to break in a minor) or `@stable` (requires major + deprecation cycle)? See `doc/api-management.md`. |
| R5 | EPL obligations understood? (see `knowledge/04-epl-guide.md`) |

---

## Phase 5 — Feature Design Document

Fill in this template. Do not leave any field blank or with a placeholder — if something is genuinely unknown, mark it as `[BLOCKER — needs resolution]` and do not proceed until resolved.

```markdown
## Feature Design Document: <Feature Name>

**Date:** <YYYY-MM-DD>
**Status:** DRAFT

---

### Problem Statement
<One paragraph. What pain point? Who is affected? Why now?>

### Acceptance Criteria
1. <Specific, testable — what can you DO after this exists>
2. <Specific, testable>
3. <Specific, testable>

### Theia Extension Point(s)
- `<ContributionInterface>` from `<@theia/package>`

### Package Placement
- [ ] New package: `packages/<package-name>/`
- [ ] Existing package: `packages/<package-name>/`

**Platform targets:** [ ] browser  [ ] node  [ ] both

### Package Structure
packages/<package-name>/
├── src/
│   ├── common/   → <interfaces, shared types — or "not needed">
│   ├── browser/  → <frontend module, contributions>
│   └── node/     → <backend service — or "not needed">

### DI Binding Sketch
bind(<ServiceClass>).toSelf().inSingletonScope();
bind(<ContributionInterface>).toService(<ServiceClass>);

### Dependencies
| Package | Reason |
|---|---|
| `@theia/core` | base DI, commands, menus |

### Core Theia Files Modified
NONE — all via extension points.

### Codebase Search Results
- Terms searched: <list>
- Equivalent found: No / Yes — <describe>

### Risks & Open Questions
- <item>

---
**Status:** [ ] FDD confirmed by user → proceed to `skills/create-theia-extension.md`
```

---

## Phase 6 — User Confirmation Gate

Present the FDD and ask:

> "Does this match your vision? Any corrections before I hand this to the coder?"

**Do not proceed until the user says "Go ahead", "Proceed", "LGTM", or equivalent.**

If they request changes: revise the FDD, present again, repeat.

---

## Handling Resistance

If the user pushes back on planning ("just build it", "figure it out"):

1. Acknowledge: "I hear you, let's keep it tight."
2. Reduce to the minimum: ask only Q1, Q2, Q3, Q6, R1.
3. Produce a minimal FDD with `[BLOCKER]` flags for every gap.
4. State clearly: "These blockers must be resolved before I can write correct code."

---

## Next Steps After Confirmed FDD

| Feature type | Proceed to |
|---|---|
| New Theia extension/package | `skills/create-theia-extension.md` |
| New AI agent | `skills/create-ai-agent.md` |
| Theme / look & feel | `skills/customize-look-and-feel.md` |
| Multiple types combined | Run all relevant skills in sequence |

Implementation should be done with the `theia-coder.agent.md` context loaded and `instructions/extension-coding.instructions.md` active.
