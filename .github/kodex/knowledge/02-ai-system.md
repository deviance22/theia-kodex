# 02 — Theia AI Subsystem

> Everything you need to know to use, extend, and build on top of Theia's built-in AI system.

---

## 1. System Overview

```
User Input (Chat UI / Inline / Code Completion)
        │
        ▼
  ChatService  ──── manages ChatSessions (history, state)
        │
        ▼
  Agent  (implements the Agent interface)
    ├── PromptService  ──── resolves prompt templates + variables
    ├── ToolInvocationRegistry  ──── available function calls (tools)
    └── LanguageModelRegistry
              │
              ▼
        LanguageModelProvider
         (OpenAI / Anthropic / Ollama / Hugging Face / MCP / ...)
              │
              ▼
        Streaming LLM Response
              │
              ▼
    ChatResponseModel → streamed to UI via ChatModel
```

**Key packages:**

| Package | Responsibility |
|---|---|
| `@theia/ai-core` | Core interfaces: Agent, LanguageModel, PromptService, ToolRegistry, VariableService |
| `@theia/ai-chat` | ChatService, ChatSession, ChatModel, ChatAgent base |
| `@theia/ai-chat-ui` | Chat panel UI, message rendering, input widget |
| `@theia/ai-ide` | Built-in agents: Coder, Architect, Explore, CodeReviewer, AppTester |
| `@theia/ai-openai` | OpenAI language model provider |
| `@theia/ai-anthropic` | Anthropic (Claude) provider |
| `@theia/ai-ollama` | Local Ollama provider |
| `@theia/ai-mcp` | Model Context Protocol — connect external tool servers |
| `@theia/ai-code-completion` | Inline code completion (ghost text) |
| `@theia/ai-editor` | Editor-contextual AI features |
| `@theia/ai-terminal` | AI-powered terminal assistance |
| `@theia/ai-history` | Conversation history persistence |

---

## 2. Core Interfaces

### `Agent`

```ts
// packages/ai-core/src/common/agent.ts
export interface Agent {
    readonly id: string;           // unique identifier (= name currently)
    readonly name: string;         // human-readable, shown in UI
    readonly description: string;  // markdown, shown in AI config view
    readonly variables: string[];  // global variables always available
    readonly prompts: PromptVariantSet[];  // prompt templates this agent uses
    readonly languageModelRequirements: LanguageModelRequirement[];
    readonly tags?: string[];
    invoke(request: MutableChatRequestModel): Promise<void>;
}
```

### `LanguageModel`

```ts
export interface LanguageModel {
    readonly id: string;
    readonly providerId: string;
    request(params: LanguageModelRequestParameters, token?: CancellationToken): Promise<LanguageModelResponse>;
}
```

### `PromptService`

Resolves Handlebars-like templates with `{{variableName}}` and `~{functionName}` syntax:

```ts
export interface PromptService {
    getPrompt(id: string, context?: AIVariableContext): Promise<ResolvedPromptTemplate | undefined>;
    getAllPrompts(): PromptFragment[];
    storePromptTemplate(fragment: BasePromptFragment): void;
}
```

### `ToolInvocationRegistry`

```ts
export interface ToolInvocationRegistry {
    registerTool(tool: ToolRequest): void;
    getFunction(toolId: string): ToolRequest | undefined;
    getAllFunctions(): ToolRequest[];
}
```

### `AIVariableService`

Resolves `{{variableName}}` inside prompt templates at runtime:

```ts
export interface AIVariableService {
    registerVariable(contribution: AIVariableContribution): void;
    resolveVariable(request: AIVariableRequest, context: AIVariableContext): Promise<ResolvedAIVariable | undefined>;
}
```

---

## 3. Built-in Agents (`@theia/ai-ide`)

| Agent | ID | Purpose |
|---|---|---|
| `CoderAgent` | `Coder` | Full-workspace code editing, file changes, agent mode |
| `ArchitectAgent` | `Architect` | System design, planning, non-code-editing tasks |
| `ExploreAgent` | `Explore` | Codebase exploration and explanation |
| `CodeReviewerAgent` | `CodeReviewer` | PR review, code quality feedback |
| `AppTesterAgent` | `AppTester` | UI test generation and execution |
| `ProjectInfoAgent` | `ProjectInfo` | Project context gathering |
| `CreateSkillAgent` | `CreateSkill` | Generate new skill prompt templates |
| `GithubChatAgent` | `Github` | GitHub issues, PRs integration |

Each agent:
- Is decorated `@injectable()`
- Implements `Agent` (usually via `AbstractChatAgent` or `AbstractModeAwareChatAgent`)
- Declares its `prompts` (template IDs)
- Declares its `languageModelRequirements`
- Is bound in the frontend module: `bind(Agent).toService(CoderAgent)`

---

## 4. Creating a Custom AI Agent

### Step 1 — Define the Agent Class

```ts
// src/browser/my-agent.ts
// *****************************************************************************
// Copyright (C) 2025 Your Company.
// SPDX-License-Identifier: EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0
// *****************************************************************************
import { injectable, inject } from '@theia/core/shared/inversify';
import { AbstractStreamParsingChatAgent, SystemMessageDescription } from '@theia/ai-chat/lib/common';
import { LanguageModelRequirement, PromptVariantSet } from '@theia/ai-core';
import { nls } from '@theia/core';
import { MY_AGENT_SYSTEM_PROMPT_ID } from './my-agent-prompt-template';

export const MyAgentId = 'MyAgent';

@injectable()
export class MyAgent extends AbstractStreamParsingChatAgent {
    readonly id = MyAgentId;
    readonly name = MyAgentId;
    readonly description = nls.localize('theia/my-org/myAgent/description', 'My custom AI agent description.');

    readonly languageModelRequirements: LanguageModelRequirement[] = [{
        purpose: 'chat',
        identifier: 'default/code'
    }];

    readonly prompts: PromptVariantSet[] = [{
        id: MY_AGENT_SYSTEM_PROMPT_ID,
        defaultVariant: {
            id: MY_AGENT_SYSTEM_PROMPT_ID,
            template: 'You are a helpful assistant for {{productName}}. {{today}}'
        }
    }];

    protected override defaultLanguageModelPurpose = 'chat';

    protected override async getSystemMessageDescription(): Promise<SystemMessageDescription | undefined> {
        const prompt = await this.promptService.getPrompt(MY_AGENT_SYSTEM_PROMPT_ID);
        if (!prompt) { return undefined; }
        return SystemMessageDescription.fromResolvedPromptTemplate(prompt);
    }
}
```

### Step 2 — Bind in the Frontend Module

```ts
// src/browser/my-feature-frontend-module.ts
import { bindRootContributionProvider } from '@theia/core';
import { ContainerModule } from '@theia/core/shared/inversify';
import { Agent } from '@theia/ai-core';
import { MyAgent } from './my-agent';

export default new ContainerModule(bind => {
    bind(MyAgent).toSelf().inSingletonScope();
    bind(Agent).toService(MyAgent);
});
```

---

## 5. Creating a Custom Tool (Function Call)

```ts
// src/browser/my-tool.ts
import { injectable } from '@theia/core/shared/inversify';
import { ToolProvider, ToolRequest } from '@theia/ai-core';

@injectable()
export class MyTool implements ToolProvider {
    getTool(): ToolRequest {
        return {
            id: 'my_org_get_data',
            name: 'my_org_get_data',
            description: 'Retrieves data from My Service. Returns a list of items.',
            parameters: {
                type: 'object',
                properties: {
                    query: {
                        type: 'string',
                        description: 'Search query'
                    }
                },
                required: ['query']
            },
            handler: async (args: string) => {
                const { query } = JSON.parse(args) as { query: string };
                // your implementation
                return JSON.stringify({ items: [`Result for ${query}`] });
            }
        };
    }
}
```

Bind in module:

```ts
import { ToolProvider } from '@theia/ai-core';
import { bindToolProvider } from '@theia/ai-core/lib/common/tool-invocation-registry';

// In ContainerModule:
bindToolProvider(MyTool, bind);
```

---

## 6. Creating a Prompt Variable

```ts
// src/browser/my-variable-contribution.ts
import { injectable, inject } from '@theia/core/shared/inversify';
import { AIVariableContribution, AIVariableService, ResolvedAIVariable, AIVariableContext, AIVariable } from '@theia/ai-core';

export const MY_VARIABLE: AIVariable = {
    id: 'myVariable',
    name: 'myVariable',
    description: 'The current project name'
};

@injectable()
export class MyVariableContribution implements AIVariableContribution {
    readonly variable = MY_VARIABLE;

    async resolve(variable: AIVariable, args: string | undefined, context: AIVariableContext): Promise<ResolvedAIVariable | undefined> {
        const value = 'MyProject'; // compute actual value
        return { variable, value };
    }
}
```

Bind in module:

```ts
import { AIVariableContribution } from '@theia/ai-core';
bind(MyVariableContribution).toSelf().inSingletonScope();
bind(AIVariableContribution).toService(MyVariableContribution);
```

---

## 7. Prompt Templates

Prompt templates use a Handlebars-like syntax with Theia extensions:

```handlebars
---
id: my-agent-system-prompt
name: My Agent System Prompt
description: System prompt for My Agent
---
You are a coding assistant for {{productName}}.

Today is {{today}}.

The current workspace contains these files:
{{#each files}}
- {{this}}
{{/each}}

Available tools:
~{my_org_get_data}
```

- `{{variableName}}` — resolved by `AIVariableService`
- `~{toolName}` — includes tool definition in the prompt
- Front matter (`---`) provides metadata (name, description, isCommand flag)

---

## 8. Language Model Providers

To add a new LLM backend:

```ts
import { LanguageModelProvider, AbstractLanguageModel } from '@theia/ai-core';
import { injectable } from '@theia/core/shared/inversify';

@injectable()
export class MyLLMProvider implements LanguageModelProvider {
    async getLanguageModels(): Promise<AbstractLanguageModel[]> {
        return [new MyLanguageModel('my-model-id', 'my-model-name')];
    }
}

// Bind in module:
bind(MyLLMProvider).toSelf().inSingletonScope();
bind(LanguageModelProvider).toService(MyLLMProvider);
```

---

## 9. MCP (Model Context Protocol)

Theia has first-class MCP support via `@theia/ai-mcp`. MCP servers expose tools over stdio/SSE that agents can invoke like any other tool.

Configuration is done via preferences:

```json
{
  "ai.mcp.servers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allow"]
    }
  }
}
```

Once configured, MCP tools are automatically registered in `ToolInvocationRegistry` and available to all agents.

---

## 10. AI Configuration UI

The built-in AI Configuration View (`@theia/ai-core-ui`) provides:
- Agent enable/disable toggles
- Language model assignment per agent
- Prompt template customization (override system prompts)
- Variable inspection

Users can customize any agent's system prompt via the UI without code changes.
