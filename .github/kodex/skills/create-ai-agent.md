# Skill: Create a New Custom AI Agent

> **STOP — Planning Gate**
> This skill requires a completed, user-confirmed Feature Design Document (FDD) before any files are created.
> If you do not have one, run `skills/feature-planning.md` first.
> Load `agents/theia-planner.agent.md` to conduct the planning session.

**When to use:** After a confirmed FDD — the user wants to add a new AI agent to their Theia-based IDE.

**When NOT to use:** No confirmed FDD yet (run `skills/feature-planning.md` first); modifying an existing built-in agent (use override via DI rebinding instead); adding a tool to an existing agent.

---

## Workflow

### 1. Gather Requirements

Ask the user:
1. **Agent name** — PascalCase, unique, concise. No "Agent" or "Chat" suffix (those are added by the UI).
2. **Purpose** — what does this agent do? What is its domain/specialty?
3. **LLM requirement** — `default/code` for coding tasks, `default/fast` for quick responses, or a specific model ID.
4. **Tools needed** — what function calls will it use? Existing tools (workspace, github, etc.) or new ones?
5. **Variables needed** — what context does it need (workspace files, current editor, today's date)?
6. **Modes** — does it need multiple prompt variants (e.g., edit mode vs. agent mode)?

### 2. File Structure to Create

```
packages/<your-pkg>/src/browser/
├── <agent-name>-agent.ts              ← agent class
├── <agent-name>-prompt-template.ts   ← system prompt definition
└── frontend-module.ts                 ← updated to bind agent

packages/<your-pkg>/src/browser/tools/  (optional)
└── <agent-name>-tool.ts               ← if new tools needed
```

### 3. Generate the Prompt Template

```ts
// src/browser/<agent-name>-prompt-template.ts
// *****************************************************************************
// Copyright (C) 2025 <Your Company>.
// SPDX-License-Identifier: EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0
// *****************************************************************************

export const <AGENT_NAME>_SYSTEM_PROMPT_ID = '<agent-name>-system-prompt';

export function get<AgentName>PromptTemplate() {
    return {
        id: <AGENT_NAME>_SYSTEM_PROMPT_ID,
        template: `---
id: ${<AGENT_NAME>_SYSTEM_PROMPT_ID}
name: <AgentName> System Prompt
description: System prompt for <AgentName>
---
You are <description of the agent's role and personality>.

You are integrated into {{productName}}, a developer IDE.

Today is {{today}}.

## Your Capabilities
- <capability 1>
- <capability 2>

## Guidelines
- Be concise and precise
- Focus on <domain>
- When unsure, ask for clarification

{{#if tools}}
## Available Tools
~{<tool_name>}
{{/if}}
`
    };
}
```

### 4. Generate the Agent Class

```ts
// src/browser/<agent-name>-agent.ts
// *****************************************************************************
// Copyright (C) 2025 <Your Company>.
// SPDX-License-Identifier: EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0
// *****************************************************************************
import { injectable } from '@theia/core/shared/inversify';
import { AbstractStreamParsingChatAgent, SystemMessageDescription } from '@theia/ai-chat/lib/common';
import { LanguageModelRequirement, PromptVariantSet } from '@theia/ai-core';
import { nls } from '@theia/core';
import { <AGENT_NAME>_SYSTEM_PROMPT_ID, get<AgentName>PromptTemplate } from './<agent-name>-prompt-template';

export const <AgentName>AgentId = '<AgentName>';

@injectable()
export class <AgentName>Agent extends AbstractStreamParsingChatAgent {
    readonly id = <AgentName>AgentId;
    readonly name = <AgentName>AgentId;

    readonly description = nls.localize(
        'theia/<your-pkg>/<agentName>/description',
        '<Plain-language description shown in the AI configuration view.>'
    );

    readonly tags = ['<domain>'];  // optional: filters agents in the UI

    readonly languageModelRequirements: LanguageModelRequirement[] = [{
        purpose: 'chat',
        identifier: 'default/code'   // or 'default/fast' for lightweight tasks
    }];

    readonly prompts: PromptVariantSet[] = [{
        id: <AGENT_NAME>_SYSTEM_PROMPT_ID,
        defaultVariant: get<AgentName>PromptTemplate()
    }];

    protected override defaultLanguageModelPurpose = 'chat';

    protected override async getSystemMessageDescription(): Promise<SystemMessageDescription | undefined> {
        const prompt = await this.promptService.getPrompt(<AGENT_NAME>_SYSTEM_PROMPT_ID);
        if (!prompt) { return undefined; }
        return SystemMessageDescription.fromResolvedPromptTemplate(prompt);
    }
}
```

### 5. Add Custom Tools (if needed)

```ts
// src/browser/tools/<agent-name>-tool.ts
// *****************************************************************************
// Copyright (C) 2025 <Your Company>.
// SPDX-License-Identifier: EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0
// *****************************************************************************
import { injectable } from '@theia/core/shared/inversify';
import { ToolProvider, ToolRequest } from '@theia/ai-core';

@injectable()
export class <AgentName>Tool implements ToolProvider {
    getTool(): ToolRequest {
        return {
            id: '<your_org>_<tool_name>',
            name: '<your_org>_<tool_name>',
            description: '<Precise description of what this tool does and returns. The LLM reads this.>',
            parameters: {
                type: 'object',
                properties: {
                    input: {
                        type: 'string',
                        description: 'The input value'
                    }
                },
                required: ['input']
            },
            handler: async (argsJson: string): Promise<string> => {
                try {
                    const { input } = JSON.parse(argsJson) as { input: string };
                    if (!input) {
                        return JSON.stringify({ error: 'input is required' });
                    }
                    // implementation
                    return JSON.stringify({ result: `Processed: ${input}` });
                } catch (e) {
                    return JSON.stringify({ error: 'Failed to process request' });
                }
            }
        };
    }
}
```

### 6. Update the Frontend Module

```ts
// Add to your existing ContainerModule:
import { bindRootContributionProvider } from '@theia/core';
import { Agent, ToolProvider } from '@theia/ai-core';
import { bindToolProvider } from '@theia/ai-core/lib/common/tool-invocation-registry';
import { <AgentName>Agent } from './<agent-name>-agent';
import { <AgentName>Tool } from './tools/<agent-name>-tool';

export default new ContainerModule(bind => {
    // Existing bindings...

    // Register the agent
    bind(<AgentName>Agent).toSelf().inSingletonScope();
    bind(Agent).toService(<AgentName>Agent);

    // Register custom tools (if any)
    bindToolProvider(<AgentName>Tool, bind);
});
```

### 7. Verify

```bash
npm run compile       # TypeScript compiles cleanly
npm run lint          # lint passes
npm run build:browser # bundle includes the new agent
npm run start:browser # open http://localhost:3000 → AI Chat → agent appears in selector
```

---

## Checklist

- [ ] Agent `id` and `name` are identical and unique in the IDE
- [ ] `description` uses `nls.localize` with `theia/<pkg>/<agent>/description` key
- [ ] Prompt template has correct front matter `id`
- [ ] `languageModelRequirements` declares at least one requirement
- [ ] `prompts` array is populated with the template
- [ ] Agent bound: `bind(Agent).toService(<AgentName>Agent)`
- [ ] Tools bound via `bindToolProvider`
- [ ] Tool handlers validate and JSON.parse inside try/catch
- [ ] SPDX headers on all new files
- [ ] Agent appears in AI Configuration view after start
