# Skill: Create a New Theia Extension Package

> **STOP — Planning Gate**
> This skill requires a completed, user-confirmed Feature Design Document (FDD) before any files are created.
> If you do not have one, run `skills/feature-planning.md` first.
> Load `agents/theia-planner.agent.md` to conduct the planning session.

**When to use:** After a confirmed FDD — the user wants to scaffold a new Theia package from scratch (a new feature module, a new extension, a new service).

**When NOT to use:** No confirmed FDD yet (run `skills/feature-planning.md` first); adding a single file or class to an existing package; creating an AI agent (use `create-ai-agent.md` instead).

---

## Workflow

### 1. Gather Requirements

Ask the user:
1. **Package name** — what is the `@my-org/package-name`? (kebab-case)
2. **Purpose** — what does this package do?
3. **Platform targets** — does it need a backend (Node.js)? Frontend only? Both?
4. **Dependencies** — what Theia packages does it depend on? (usually at least `@theia/core`)
5. **Contribution points** — what does it contribute? (commands, menus, widgets, AI agents, etc.)

### 2. Create the Package Directory Structure

```
packages/<package-name>/
├── package.json
├── tsconfig.json
└── src/
    ├── browser/
    │   ├── <package-name>-frontend-module.ts
    │   └── <package-name>-contribution.ts       (if contributing commands/menus)
    ├── common/
    │   └── <package-name>-service.ts             (shared interfaces, if needed)
    └── node/
        ├── <package-name>-backend-module.ts      (if backend needed)
        └── <package-name>-service-impl.ts        (if backend needed)
```

### 3. Generate `package.json`

```json
{
  "name": "@my-org/<package-name>",
  "version": "1.0.0",
  "description": "<description>",
  "license": "EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0",
  "theiaExtensions": [{
    "frontend": "lib/browser/<package-name>-frontend-module",
    "backend":  "lib/node/<package-name>-backend-module"
  }],
  "main": "lib/browser/index",
  "scripts": {
    "prepare": "theiaext prepare",
    "clean": "theiaext clean",
    "build": "theiaext build",
    "compile": "theiaext compile",
    "watch": "theiaext watch",
    "lint": "theiaext lint",
    "test": "theiaext test"
  },
  "dependencies": {
    "@theia/core": "^1.71.0"
  },
  "devDependencies": {
    "@theia/ext-scripts": "^1.71.0"
  }
}
```

Remove the `"backend"` entry from `theiaExtensions` if no backend is needed.

### 4. Generate `tsconfig.json`

```json
{
  "extends": "../../configs/base.tsconfig.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "lib"
  },
  "include": ["src"]
}
```

### 5. Generate the Frontend Module

```ts
// src/browser/<package-name>-frontend-module.ts
// *****************************************************************************
// Copyright (C) 2025 <Your Company>.
//
// This program and the accompanying materials are made available under the
// terms of the Eclipse Public License v. 2.0 which is available at
// http://www.eclipse.org/legal/epl-2.0.
//
// SPDX-License-Identifier: EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0
// *****************************************************************************
import { CommandContribution, MenuContribution } from '@theia/core';
import { ContainerModule } from '@theia/core/shared/inversify';
import { <PascalName>Contribution } from './<package-name>-contribution';

export default new ContainerModule(bind => {
    bind(<PascalName>Contribution).toSelf().inSingletonScope();
    bind(CommandContribution).toService(<PascalName>Contribution);
    bind(MenuContribution).toService(<PascalName>Contribution);
});
```

### 6. Generate the Contribution Class

```ts
// src/browser/<package-name>-contribution.ts
// *****************************************************************************
// Copyright (C) 2025 <Your Company>.
// SPDX-License-Identifier: EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0
// *****************************************************************************
import { Command, CommandContribution, CommandRegistry, MenuContribution, MenuModelRegistry } from '@theia/core';
import { injectable } from '@theia/core/shared/inversify';
import { nls } from '@theia/core';

export const <SCREAMING_SNAKE>_COMMAND: Command = {
    id: '<package-name>:action',
    label: '<Label>'
};

@injectable()
export class <PascalName>Contribution implements CommandContribution, MenuContribution {
    registerCommands(registry: CommandRegistry): void {
        registry.registerCommand(<SCREAMING_SNAKE>_COMMAND, {
            execute: () => this.execute()
        });
    }

    registerMenus(menus: MenuModelRegistry): void {
        // Add to appropriate menu group
    }

    protected execute(): void {
        // implementation
    }
}
```

### 7. Register in Your App's `package.json`

In your application's `package.json` (e.g., `examples/browser/package.json`):

```json
"dependencies": {
    "@my-org/<package-name>": "^1.0.0"
}
```

### 8. After Creation

```bash
npm install          # resolves dependencies + regenerates TypeScript project references
npm run compile      # verify TypeScript compiles cleanly
npm run lint         # verify lint passes
```

---

## Checklist

- [ ] `package.json` has correct name, license, theiaExtensions
- [ ] `tsconfig.json` extends `../../configs/base.tsconfig.json`
- [ ] All new files have SPDX copyright header
- [ ] No imports crossing platform boundaries
- [ ] All singletons use `inSingletonScope()`
- [ ] `bindRootContributionProvider` used if registering contribution providers
- [ ] User-facing strings localized
- [ ] Added to app's package.json dependencies
