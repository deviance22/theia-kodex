---
applyTo: "**"
description: "Mandatory planning/grill mode gate — ALWAYS enter planning mode before creating any new feature, package, extension, or theme."
---

# Planning & Grill Mode — Safety Net

> **This instruction applies to every creation request. No exceptions.**

## Rule: Plan Before You Build

Whenever a user asks to:

- Create a new package or extension
- Add a new feature, widget, command, or menu item
- Add a new AI agent, tool, or variable
- Change the look and feel (new theme, color token, style)
- Scaffold anything new in the codebase

**You MUST enter Planning Mode first.**

Do NOT write any code, create any files, or propose any directory structures until Planning Mode is complete and the user has confirmed they have a clear picture.

---

## Planning Mode Protocol

### Phase 1 — Grill Mode

Ask questions grouped by category. Do not ask all at once — open with the Core set, then drill into the relevant Technical set based on answers. Mark each category complete as you get clear answers.

#### Core Questions (always ask first)

1. **What problem does this solve?** Describe the pain point or missing capability from the end user's perspective.
2. **Who is the user?** Developer using the IDE? End user of an application? A background service?
3. **What does done look like?** Give me two or three concrete acceptance criteria — what can you do after this exists that you can't do now?
4. **Scope:** Is this a new package or an addition to an existing package?
5. **Priority:** Is this blocking something else, or exploratory?

#### Technical Questions (ask after Core is clear)

6. **Platform:** Frontend only, backend only, or both? Does it need a WebSocket/RPC link between the two?
7. **Extension point:** Which Theia contribution interface does this use? (CommandContribution, MenuContribution, Agent, ToolProvider, ColorContribution, StylingParticipant, FrontendApplicationContribution, etc.) — search the knowledge base if unsure.
8. **Existing equivalent:** Have we checked whether Theia already provides this? Which packages were searched?
9. **State & persistence:** Does this need to remember anything? (UserStorageService, PreferenceService, StorageService)
10. **Dependencies:** Which `@theia/*` packages are needed? Any third-party npm packages?

#### AI-Specific Questions (only if the feature involves AI)

11. **New or extended agent?** Is this a brand-new agent or an extension of an existing one?
12. **Tools:** What function calls does the agent need? (filesystem, terminal, search, custom)
13. **Variables:** What context must be injected into the prompt? (open files, project structure, current selection)
14. **LLM requirement:** Any specific model requirement, or use the workspace default?

#### Look & Feel Questions (only if the feature involves UI/theming)

15. **Scope of change:** Application-wide branding, one component, or a full new theme?
16. **Dark/light/high-contrast:** Are all three variants accounted for?
17. **Existing token or new?** Is this overriding an existing `--theia-*` CSS variable or registering a new semantic color?
18. **Accessibility:** Has WCAG 2.2 AA contrast (4.5:1 text, 3:1 UI components) been considered for all variants?

#### Risk Questions (always ask last)

19. **Core files:** Does any part of this require modifying a file inside `packages/core/` or any other existing Theia package? If yes, why can't an extension point be used instead?
20. **EPL obligations:** Is any third-party code being copied in? What is its license?
21. **Breaking changes:** Could this change an existing public API surface? If yes — is the API tagged `@experimental` (minor-safe to break) or `@stable` (requires major release + deprecation cycle)? See `doc/api-management.md`.

---

### Phase 2 — Knowledge Base & Codebase Validation

Before declaring Planning Mode complete, verify the following using the knowledge base and a codebase search:

- [ ] The correct extension point has been identified (cross-check `knowledge/01-extension-development.md` or `knowledge/02-ai-system.md`)
- [ ] No equivalent already exists in `packages/` (run a grep/semantic search to confirm)
- [ ] The package placement is correct (new package vs. existing package vs. product repo)
- [ ] EPL-2.0 obligations are understood (`knowledge/04-epl-guide.md`)
- [ ] The theming approach is decided if any UI is involved (`knowledge/06-look-and-feel.md`)

---

### Phase 3 — Feature Design Document (FDD)

Planning Mode is complete only when you can produce — and the user **confirms** — a Feature Design Document with all sections filled. Do not proceed without explicit user confirmation.

```
## Feature Design Document: <Feature Name>

### Problem Statement
<One paragraph describing the pain point and the user it affects.>

### Acceptance Criteria
1. <concrete, testable criterion>
2. <concrete, testable criterion>
3. <concrete, testable criterion>

### Theia Extension Point
<Which contribution interface(s) this uses and from which @theia/* package.>

### Package Structure
<new package name OR existing package to modify>
packages/<name>/
├── src/common/   → <what goes here>
├── src/browser/  → <what goes here>
└── src/node/     → <what goes here, or "not needed">

### DI Binding Sketch
<High-level list of bind() calls — not full code yet, just symbols and scopes.>

### Dependencies
<@theia/* packages + any third-party npm packages with their licenses.>

### Core Files Modified
NONE — uses extension points.
OR: <list any files that must change, with justification and upstream issue link.>

### Risks & Open Questions
- <risk or unknown item>
```

---

## Satisfaction Criteria

You are allowed to proceed to implementation **only when all of the following are true**:

1. All Core + relevant Technical/AI/Look & Feel + Risk questions have clear, non-vague answers.
2. A codebase search has confirmed no equivalent exists.
3. The FDD above is filled in with no placeholders.
4. The user has explicitly said: **"Proceed"** or **"Go ahead"** (or equivalent confirmation).

If the user pushes back on questions ("just do it", "figure it out"), respond:

> "I understand the urgency — I just need 2 more minutes of your time on these questions to avoid building the wrong thing. A misaligned implementation costs far more to fix than planning does."

---

## Reference Files

| Topic                           | Knowledge file                          |
| ------------------------------- | --------------------------------------- |
| Architecture & extension points | `knowledge/00-architecture.md`          |
| Building extensions             | `knowledge/01-extension-development.md` |
| AI agents & tools               | `knowledge/02-ai-system.md`             |
| EPL-2.0 compliance              | `knowledge/04-epl-guide.md`             |
| Theming & look & feel           | `knowledge/06-look-and-feel.md`         |
| Scaffold a package              | `skills/create-theia-extension.md`      |
| Scaffold an AI agent            | `skills/create-ai-agent.md`             |
| Customize styling               | `skills/customize-look-and-feel.md`     |
