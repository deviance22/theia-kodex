# 00 — Theia Architecture Breakdown

> Audience: Engineers new to the project. Read this before writing any code.

---

## 1. What is Eclipse Theia?

Theia is an open, flexible, **extensible IDE platform** built with web technologies.
It is **not** an IDE itself — it is a framework for building IDEs and cloud tools.
The `theia-kodex` repository contains Theia's source, which you use as a dependency when you ship your product.

Key facts:
- Written entirely in **TypeScript** (target ES2023, CommonJS modules)
- Runs in the **browser** (server + browser client), **Electron** (desktop app), and **browser-only** mode
- Uses **InversifyJS** for Dependency Injection throughout
- Uses **Monaco Editor** (same engine as VS Code) for code editing
- Uses **Lumino 2.x** for the widget/panel layout system
- Supports VS Code extensions at runtime via the plugin API

---

## 2. Monorepo Structure

```
theia-kodex/
├── packages/         ← 77+ runtime packages (each is its own npm package)
│   ├── core/         ← foundational: DI, commands, menus, shell, messaging
│   ├── editor/       ← Monaco editor integration
│   ├── filesystem/   ← virtual filesystem
│   ├── terminal/     ← terminal widget
│   ├── debug/        ← DAP debug support
│   ├── ai-core/      ← AI infrastructure: agents, LLMs, prompts, tools
│   ├── ai-chat/      ← chat model, sessions, agents
│   ├── ai-chat-ui/   ← chat UI widgets
│   ├── ai-ide/       ← built-in agents (Coder, Architect, Explore, etc.)
│   ├── ai-openai/    ← OpenAI provider
│   ├── ai-anthropic/ ← Anthropic provider
│   ├── ai-ollama/    ← local Ollama provider
│   ├── ai-mcp/       ← Model Context Protocol integration
│   └── ...           ← 60+ more packages
├── dev-packages/     ← build tooling (not shipped to users)
│   ├── application-manager/  ← Theia CLI / app generator
│   ├── cli/                  ← `theia` CLI command
│   └── private-ext-scripts/  ← `theiaext` build wrapper
├── examples/         ← runnable reference applications
│   ├── browser/      ← browser app (all packages, AI-enabled)
│   ├── electron/     ← Electron desktop app
│   └── browser-only/ ← no Node.js backend
├── configs/          ← shared tsconfig, eslint, mocha, nyc configs
└── doc/              ← authoritative documentation
```

---

## 3. Platform Folders (per package)

Each package in `packages/` organises its source by **runtime target**:

| Folder | Runs on | Can import from |
|---|---|---|
| `src/common/` | everywhere | nothing platform-specific |
| `src/browser/` | browser frontend | `common` |
| `src/browser-only/` | browser, no Node backend | `common` |
| `src/node/` | Node.js backend | `common` |
| `src/electron-browser/` | Electron renderer | `common`, `browser` |
| `src/electron-main/` | Electron main process | `common`, `node`, `electron-node` |
| `src/electron-node/` | Electron Node side | `common`, `node` |

**Rule:** Never import `browser` code from `node`, or vice-versa. Lint enforces this.

---

## 4. Extension Entry Points

Each package exports its DI modules via `package.json`:

```json
"theiaExtensions": [{
  "frontend": "lib/browser/my-frontend-module",
  "backend":  "lib/node/my-backend-module"
}]
```

The `@theia/application-manager` reads these during build to assemble the final application.

---

## 5. Dependency Injection (InversifyJS)

Theia's entire architecture is built on InversifyJS. Key rules:

```ts
// Every managed class is decorated:
@injectable()
export class MyService {
    // Property injection (preferred — constructor injection is a breaking change):
    @inject(OtherService)
    protected readonly other: OtherService;

    // Initialization goes here, not the constructor:
    @postConstruct()
    protected init(): void {
        this.other.doSomething();
    }
}

// Binding in a ContainerModule:
export default new ContainerModule(bind => {
    bind(MyService).toSelf().inSingletonScope();   // always inSingletonScope for singletons
});
```

**Common pitfalls:**
- Forgetting `inSingletonScope()` → new instance per injection
- Using `bindContributionProvider` instead of `bindRootContributionProvider` in top-level modules → memory leak
- Doing work in the constructor instead of `@postConstruct`

---

## 6. Contribution Points Pattern

Theia uses a **contribution points** pattern for extensibility. Instead of modifying core, you contribute to named interfaces:

| Contribution Interface | Purpose |
|---|---|
| `CommandContribution` | Register commands |
| `MenuContribution` | Add menu items |
| `KeybindingContribution` | Register keyboard shortcuts |
| `FrontendApplicationContribution` | Lifecycle hooks (onStart, onStop) |
| `TabBarToolbarContribution` | Add toolbar buttons |
| `ColorContribution` | Register theme colors |
| `LabelProviderContribution` | Customize file/URI labels |
| `Agent` | Register an AI agent |
| `LanguageModelProvider` | Register an LLM backend |
| `AIVariableContribution` | Register a prompt variable |
| `ToolProvider` | Register an AI tool/function |

Binding pattern:

```ts
// In your ContainerModule:
bind(MyCommandContribution).toSelf().inSingletonScope();
bind(CommandContribution).toService(MyCommandContribution);
```

---

## 7. Frontend ↔ Backend Communication

Services that span frontend and backend use JSON-RPC over WebSocket:

```
Browser (Frontend)
  └── Frontend Delegate (proxy) ─── WebSocket/JSON-RPC ──→ Backend Service (Node.js)
```

Pattern used in `ai-core`:
1. Define interface + path in `src/common/`
2. Implement in `src/node/`
3. Create proxy in `src/browser/` using `ServiceConnectionProvider`

```ts
// src/browser/my-frontend-module.ts
bind(MyService).toDynamicValue(ctx => {
    const provider = ctx.container.get<ServiceConnectionProvider>(RemoteConnectionProvider);
    return provider.createProxy<MyService>(MY_SERVICE_PATH);
}).inSingletonScope();
```

---

## 8. Widget System (Lumino)

Theia's layout is managed by Lumino's `DockPanel`. Key widget types:

| Type | Use case |
|---|---|
| `BaseWidget` | Simple non-React widgets |
| `ReactWidget` | React-rendered widgets |
| `TreeWidget` | Tree-view (file explorer, outline) |
| `ViewContainer` | Collapsible sections container |
| `TabBarToolbar` | Toolbar above a widget |

Widgets are registered as `WidgetFactory` bindings and opened via `OpenerService` or commands.

---

## 9. The AI Subsystem at a Glance

```
User Input
   ↓
ChatService (manages sessions, routes to agents)
   ↓
Agent (e.g. CoderAgent, ArchitectAgent)
   ├── PromptService (resolves templates + variables)
   ├── ToolInvocationRegistry (available function calls)
   └── LanguageModelRegistry
           ↓
     LanguageModelProvider (OpenAI / Anthropic / Ollama / ...)
           ↓
     LLM Response → streamed back through ChatModel → UI
```

See `knowledge/02-ai-system.md` for full detail.

---

## 10. File Naming Conventions

| Convention | Example |
|---|---|
| kebab-case filenames | `my-service.ts` |
| Filename = main exported type | `terminal-widget.ts` exports `TerminalWidget` |
| Unit tests | `my-service.spec.ts` |
| UI tests | `my-service.ui-spec.ts` |
| Slow/integration tests | `my-service.slow-spec.ts` |
| Test helpers | `src/node/test/test-helper.ts` |

---

## 11. Key Config Files

| File | Purpose |
|---|---|
| `configs/base.tsconfig.json` | TypeScript base (all packages extend this) |
| `configs/base.eslintrc.json` | ESLint parser and base rules |
| `configs/build.eslintrc.json` | ESLint build rules |
| `configs/mocharc.yml` | Mocha test runner config |
| `configs/nyc.json` | Istanbul coverage config |
| `lerna.json` | Monorepo package graph |
| `package.json` (root) | Root scripts (`build:browser`, `start:browser`, etc.) |

---

## 12. Technology Stack Summary

| Layer | Technology |
|---|---|
| Language | TypeScript ~5.9.3 |
| Runtime (backend) | Node.js ≥ 20 |
| Runtime (frontend) | Browser (Chrome-class) |
| UI framework | React 18.2.0 + Lumino 2.x |
| Code editor | Monaco Editor |
| DI | InversifyJS |
| Build | Lerna + Webpack (app) + tsc (packages) |
| Test | Mocha + NYC (Istanbul) |
| Linting | ESLint (custom Theia plugin) |
