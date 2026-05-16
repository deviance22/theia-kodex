---
applyTo: "packages/ai-*/**/*.ts,src/browser/ai-*.ts,src/common/ai-*.ts,src/node/ai-*.ts"
description: "Rules for writing custom AI agents, tools, variables, and LLM providers in Theia"
---

# Theia AI Agent Coding Instructions

<!-- Applies to AI-related TypeScript files.
     Pairs with extension-coding.instructions.md — all base rules still apply. -->

## Module Bindings for AI Packages

```ts
export default new ContainerModule((bind) => {
    // Contribution providers must use bindRootContributionProvider
    bindRootContributionProvider(bind, Agent);
    bindRootContributionProvider(bind, LanguageModelProvider);

    // Bind your agent
    bind(MyAgent).toSelf().inSingletonScope();
    bind(Agent).toService(MyAgent);
});
```

## Contribution Type Registry

| What you're adding   | Bind against                                        |
| -------------------- | --------------------------------------------------- |
| New LLM backend      | `LanguageModelProvider`                             |
| Chat agent           | `Agent`                                             |
| Prompt variable      | `AIVariableContribution`                            |
| Tool / function call | `ToolProvider` via `bindToolProvider(MyTool, bind)` |

## Agent Class Template

```ts
@injectable()
export class MyAgent extends AbstractStreamParsingChatAgent {
    readonly id = "MyAgent";
    readonly name = "MyAgent"; // shown in UI — unique, no "Agent" suffix
    readonly description = nls.localize(
        "theia/my-pkg/myAgent/description",
        "What this agent does.",
    );

    readonly languageModelRequirements: LanguageModelRequirement[] = [
        {
            purpose: "chat",
            identifier: "default/code",
        },
    ];

    readonly prompts: PromptVariantSet[] = [
        {
            id: MY_SYSTEM_PROMPT_ID,
            defaultVariant: {
                id: MY_SYSTEM_PROMPT_ID,
                template: "You are a helpful assistant.",
            },
        },
    ];

    protected override defaultLanguageModelPurpose = "chat";

    protected override async getSystemMessageDescription(): Promise<
        SystemMessageDescription | undefined
    > {
        const prompt = await this.promptService.getPrompt(MY_SYSTEM_PROMPT_ID);
        if (!prompt) {
            return undefined;
        }
        return SystemMessageDescription.fromResolvedPromptTemplate(prompt);
    }
}
```

## Tool (Function Call) Template

```ts
@injectable()
export class MyTool implements ToolProvider {
    getTool(): ToolRequest {
        return {
            id: "my_org_tool_name", // snake_case, namespaced
            name: "my_org_tool_name",
            description:
                "What this tool does. Be specific — the LLM reads this.",
            parameters: {
                type: "object",
                properties: {
                    input: { type: "string", description: "The input value" },
                },
                required: ["input"],
            },
            handler: async (argsJson: string): Promise<string> => {
                const { input } = JSON.parse(argsJson) as { input: string };
                // implementation
                return JSON.stringify({ result: input });
            },
        };
    }
}

// Bind:
bindToolProvider(MyTool, bind);
```

## Prompt Variable Template

```ts
export const MY_VARIABLE: AIVariable = {
    id: "myVariable",
    name: "myVariable",
    description: "Description shown in the AI config view",
};

@injectable()
export class MyVariableContribution implements AIVariableContribution {
    readonly variable = MY_VARIABLE;

    async resolve(
        variable: AIVariable,
        args: string | undefined,
        context: AIVariableContext,
    ): Promise<ResolvedAIVariable | undefined> {
        return { variable, value: "resolved-value" };
    }
}

// Bind:
bind(MyVariableContribution).toSelf().inSingletonScope();
bind(AIVariableContribution).toService(MyVariableContribution);
```

## Prompt Templates

- Store as constants, not files (unless user-customizable)
- Use front matter for metadata
- Variables: `{{variableName}}` — resolved by AIVariableService
- Tool inclusions: `~{toolName}` — injects tool definition

```ts
export const MY_PROMPT_ID = "my-agent-system-prompt";

export function getMyPromptTemplate(): BasePromptFragment {
    return {
        id: MY_PROMPT_ID,
        template: `---
id: ${MY_PROMPT_ID}
name: My Agent System Prompt
description: System prompt for My Agent
---
You are a coding assistant for {{productName}}.

Today is {{today}}.

Use these tools when needed:
~{my_org_tool_name}
`,
    };
}
```

## Frontend-Backend Communication for AI Services

```ts
// Define path + interface in src/common/
export const MY_AI_SERVICE_PATH = "/services/my-ai-service";
export const MyAIService = Symbol("MyAIService");
export interface MyAIService {
    process(input: string): Promise<string>;
}

// Implement in src/node/
@injectable()
export class MyAIServiceImpl implements MyAIService {
    async process(input: string): Promise<string> {
        return `processed: ${input}`;
    }
}

// Proxy in src/browser/
bind(MyAIService)
    .toDynamicValue((ctx) => {
        const provider = ctx.container.get<ServiceConnectionProvider>(
            RemoteConnectionProvider,
        );
        return provider.createProxy<MyAIService>(MY_AI_SERVICE_PATH);
    })
    .inSingletonScope();
```

## Security Considerations

- **Never log API keys** — strip from any debug output or history entries
- **Validate tool inputs** — JSON.parse inside try/catch; validate required fields before use
- **Do not expose internal file paths** in tool responses — use URIs
- **Sanitize LLM outputs** before rendering in the DOM — use Theia's markdown renderer, not `innerHTML`
- **Rate-limit tool invocations** for tools that access external services

## Common Mistakes

| Mistake                             | Fix                                                   |
| ----------------------------------- | ----------------------------------------------------- |
| Agent bound but not appearing in UI | Missing `bind(Agent).toService(MyAgent)`              |
| Tool not available to agents        | Missing `bindToolProvider(MyTool, bind)`              |
| Prompt variable not resolved        | Missing `bind(AIVariableContribution).toService(...)` |
| `bindContributionProvider` used     | Use `bindRootContributionProvider`                    |
| API key hardcoded in prompt         | Read from preferences or environment variable         |
