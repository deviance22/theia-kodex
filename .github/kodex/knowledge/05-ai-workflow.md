# AI-Assisted Development Workflow for Theia-Kodex

> How to use AI tools (GitHub Copilot, built-in Theia agents) effectively when building this IDE.

---

## Overview

This project has two layers of AI:

1. **GitHub Copilot / Coding Agent** — your IDE assistant (the tool you're using right now). Guided by `.github/kodex/` files.
2. **Theia's built-in AI** — agents running inside the IDE you're building. Extensible through `@theia/ai-*` packages.

This document covers both.

---

## Layer 1: Using GitHub Copilot on This Codebase

### Recommended Workflow by Task

| Task | Load this file first | Copilot mode |
|---|---|---|
| Designing a new feature | `agents/theia-architect.agent.md` | Chat |
| Writing extension code | `agents/theia-coder.agent.md` | Agent / Edits |
| Reviewing code | `agents/theia-reviewer.agent.md` | Chat |
| Scaffolding a package | `skills/create-theia-extension.md` | Agent |
| Creating an AI agent | `skills/create-ai-agent.md` | Agent |
| Writing tests | `instructions/testing.instructions.md` | Agent / Edits |

### How to Load Context

In VS Code Copilot Chat:

```
@workspace /new feature: I want to add a terminal history search feature.
```

Or reference a specific agent:

```
Using the context from .github/kodex/agents/theia-architect.agent.md,
design the architecture for a terminal history search feature.
```

### Key Copilot Instructions Already Loaded

The following are auto-loaded via `.github/instructions/` (apply to all files):
- `theia-coding.instructions.md` — style, DI, React, i18n rules
- `theia-ai.instructions.md` — AI-specific DI and binding patterns (applies to `packages/ai-*`)

The Kodex files in `.github/kodex/` provide **product-specific** deeper context.

---

## Layer 2: Extending the Built-in IDE AI

### Built-in Agents Available in Your IDE

Once your IDE is running (`npm run start:browser`), these agents are available via the Chat panel:

| Agent | Best For |
|---|---|
| **Coder** | File editing, code generation, refactoring |
| **Architect** | System design, planning |
| **Explore** | Codebase questions, understanding code |
| **CodeReviewer** | Review files, suggest improvements |
| **AppTester** | Generate UI tests |

### Creating Product-Specific Agents

Follow `skills/create-ai-agent.md` to create agents tailored to your product domain.

Example agents you might want for a custom IDE:
- **DatabaseAgent** — generates SQL queries, explains schemas
- **DeployAgent** — builds, tests, and deploys to staging
- **DocumentationAgent** — generates and updates markdown docs
- **DebugAgent** — reads logs, identifies issues, suggests fixes

### Extending with MCP Servers

Add external tools to all agents without writing code. Example MCP servers:
- `@modelcontextprotocol/server-filesystem` — file access outside workspace
- `@modelcontextprotocol/server-github` — GitHub API
- `@modelcontextprotocol/server-postgres` — database queries
- Custom internal servers for your product's APIs

Configure via preferences:

```json
{
  "ai.mcp.servers": {
    "my-internal-api": {
      "command": "node",
      "args": ["/path/to/my-mcp-server.js"]
    }
  }
}
```

---

## Development Lifecycle with AI

### Phase 1: Feature Planning

1. Open Copilot Chat
2. Load `agents/theia-architect.agent.md` context
3. Describe the feature: what user problem does it solve?
4. Ask for: package structure, DI bindings, extension points used
5. Save the architecture decision to `doc/` or as comments in a GitHub issue

### Phase 2: Implementation

1. Scaffold with the appropriate skill file
2. Use `agents/theia-coder.agent.md` to guide code generation
3. Key prompt pattern:

   ```
   Generate the DI module for [feature]. Follow the extension-coding.instructions.md rules.
   The package is @my-org/my-feature. It contributes [commands/widgets/AI agent].
   ```

### Phase 3: Review

1. Use `agents/theia-reviewer.agent.md` for self-review before PR
2. Ask Copilot to review a specific file:

   ```
   Review packages/my-feature/src/browser/my-contribution.ts using theia-reviewer.agent.md rules.
   ```

### Phase 4: Testing

1. Use `instructions/testing.instructions.md` to write tests
2. Run: `npx lerna run test --scope @my-org/my-feature`

---

## Prompt Patterns That Work Well

### "Explain this Theia concept"

```
Explain how ContributionProvider works in Theia's DI system. 
Reference the code in packages/core/src if needed.
```

### "Scaffold from scratch"

```
Using the create-theia-extension skill, scaffold a new package @my-org/my-feature 
that contributes a command 'My Feature: Analyze File' and a sidebar widget.
```

### "Debug a DI issue"

```
My agent is bound but not showing up in the AI Configuration view.
I've done: bind(MyAgent).toSelf().inSingletonScope();
What am I missing?
```

_(Answer: `bind(Agent).toService(MyAgent)` is missing)_

### "Generate test boilerplate"

```
Generate a spec file for packages/my-feature/src/browser/my-service.ts
following the testing.instructions.md conventions.
```

---

## What AI Cannot Do Well Here

Be cautious with AI assistance for:

1. **Final DI binding decisions** — always verify that contribution providers use `bindRootContributionProvider` and singletons use `inSingletonScope()`. AI will sometimes miss this.
2. **EPL compliance** — AI cannot determine if copied code requires legal review. Always check manually.
3. **UI behavior** — the Lumino widget system has subtle layout rules. Test visually.
4. **Cross-platform edge cases** — Windows path separators, different Node.js behaviors. Test on all targets.

---

## Quick Reference Card

```
Build:    npm run build:browser
Test:     npx lerna run test --scope @my-org/pkg
Lint:     npm run lint
Start:    npm run start:browser    → http://localhost:3000
Watch:    npm run watch

New package:  follow skills/create-theia-extension.md
New agent:    follow skills/create-ai-agent.md
Architecture: load agents/theia-architect.agent.md
Code:         load agents/theia-coder.agent.md + instructions/extension-coding.instructions.md
Review:       load agents/theia-reviewer.agent.md
```
