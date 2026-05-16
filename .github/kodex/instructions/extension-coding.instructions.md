---
applyTo: "packages/**/*.ts,src/**/*.ts"
description: "Coding rules for Theia extension development: DI, platform folders, i18n, React, URIs"
---

# Theia Extension Coding Instructions

<!-- Applies to all TypeScript files in packages/ and src/ directories.
     These rules apply when writing Theia extensions and should never be violated.
     CANONICAL SOURCE: .github/instructions/theia-coding.instructions.md (auto-loaded by Copilot for all files).
     This file extends that canonical set with project-specific context. In any conflict, the canonical file wins. -->

## Style

- Single quotes for all strings; semicolons always; 4-space indentation
- `undefined` not `null`
- Explicit return types on all functions and methods
- Arrow functions preferred over anonymous function expressions
- kebab-case file names; filename matches the main exported type

## Dependency Injection

```ts
// WRONG — constructor injection breaks public API
constructor(@inject(MyService) service: MyService) {}

// CORRECT — property injection
@inject(MyService)
protected readonly service: MyService;

// CORRECT — initialization in postConstruct
@postConstruct()
protected init(): void {
    this.service.doSetup();
}
```

- Always `inSingletonScope()` for singleton bindings
- `bindRootContributionProvider` in top-level modules — `bindContributionProvider` causes memory leaks
- Never use `new` to instantiate DI-managed classes

## Platform Boundaries (enforced by ESLint)

| File location           | May import from                      |
| ----------------------- | ------------------------------------ |
| `src/common/`           | No platform APIs                     |
| `src/browser/`          | `common/` only                       |
| `src/node/`             | `common/` only                       |
| `src/electron-browser/` | `common/`, `browser/`                |
| `src/electron-main/`    | `common/`, `node/`, `electron-node/` |

## React

```ts
// WRONG — new function created on every render
<div onClick={() => this.handleClick()} />
<div onClick={this.handleClick.bind(this)} />

// CORRECT — class property arrow function
protected handleClick = (): void => { /* ... */ };
<div onClick={this.handleClick} />
```

No inline styles. No hard-coded colors. Use `var(--theia-*)` CSS variables.

## Theming & Styling

**Never hard-code hex values in TypeScript.** Route all colors through the contribution system.

Four mechanisms exist (choose the simplest one that meets the need):

| Mechanism             | Use when                                                            |
| --------------------- | ------------------------------------------------------------------- |
| CSS variable override | Tweaking layout dimensions, font, tab height                        |
| `ColorContribution`   | Adding brand colors that auto-switch with dark/light themes         |
| `StylingParticipant`  | CSS rules that differ structurally between dark/light/high-contrast |
| VS Code theme plugin  | Full color palette for the entire IDE                               |

```ts
// WRONG — hard-coded color in TypeScript
collector.addRule(".my-widget { background: #1A1A2E; }");

// CORRECT — use a registered color token
const bg = theme.getColor("brand.sidebarBackground");
if (bg) {
    collector.addRule(`.my-widget { background: ${bg}; }`);
}

// CORRECT — use a CSS variable (for colors already in the theme)
// In CSS: background: var(--theia-sideBar-background);
```

Binding a `ColorContribution`:

```ts
bind(MyColorContribution).toSelf().inSingletonScope();
bind(ColorContribution).toService(MyColorContribution);
```

Binding a `StylingParticipant`:

```ts
bind(MyStylingParticipant).toSelf().inSingletonScope();
bind(StylingParticipant).toService(MyStylingParticipant);
```

Icons must use `currentColor` in SVG fill/stroke so they adapt in forced-colors mode.
Always verify WCAG 2.2 AA contrast (≥ 4.5:1 for text) after color changes.

## URI and Path Handling

```ts
// Pass URI strings between frontend and backend — never raw paths
// Frontend → path:
const path = await this.fileService.fsPath(new URI(uriStr));
// Backend → path:
const path = FileUri.fsPath(uriStr);

// Build URIs:
const child = new URI(parent).join("subdir").toString(); // CORRECT
const child = parent + "/subdir"; // WRONG
```

## i18n

```ts
nls.localizeByDefault("Close"); // VS Code string
nls.localize("theia/my-pkg/key", "Default text"); // custom string
nls.localize("theia/my-pkg/hello", "Hello {0}!", userName); // with args

// Commands:
Command.toDefaultLocalizedCommand({ id: "my:cmd", label: "Close" });
Command.toLocalizedCommand(
    { id: "my:cmd", label: "My Label" },
    "theia/my-pkg/myCmd",
);
```

## Copyright Header (every new file)

```ts
// *****************************************************************************
// Copyright (C) 2025 <Your Company>.
//
// This program and the accompanying materials are made available under the
// terms of the Eclipse Public License v. 2.0 which is available at
// http://www.eclipse.org/legal/epl-2.0.
//
// SPDX-License-Identifier: EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0
// *****************************************************************************
```

## Contribution Binding Pattern

```ts
export default new ContainerModule((bind) => {
    bind(MyContribution).toSelf().inSingletonScope();
    bind(CommandContribution).toService(MyContribution);
    bind(MenuContribution).toService(MyContribution);
});
```

## Code Quality

Never leave:

- Commented-out code
- Changelog-style comments (`// Fixed by X on 2024-01-15`)
- Decorative dividers (`//=====`)
- Comments that restate what the code does

Only comment to explain _why_, not _what_.
