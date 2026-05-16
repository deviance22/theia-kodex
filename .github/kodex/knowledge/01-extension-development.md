# 01 — Extension Development Guide

> How to build features on top of Theia **without touching core packages**.
> Following this guide keeps your product upgradeable when Theia releases new versions.

---

## Principle: Never Fork Core

The EPL-2.0 license and practical upgrade concerns both demand the same thing:
**extend, don't modify**.

Theia is designed so that nearly every behaviour can be overridden through the DI container.
If you find yourself editing a file in `packages/core/` or another Theia package, stop —
find the contribution point or override mechanism first.

---

## 1. Creating a New Theia Package (Extension)

### Folder Structure

```
packages/my-feature/
├── package.json
├── tsconfig.json
└── src/
    ├── browser/
    │   ├── my-feature-frontend-module.ts   ← DI bindings for frontend
    │   └── my-feature-contribution.ts      ← commands, menus, etc.
    ├── common/
    │   └── my-feature-service.ts           ← shared interfaces
    └── node/
        ├── my-feature-backend-module.ts    ← DI bindings for backend
        └── my-feature-service-impl.ts      ← backend implementation
```

### Minimal `package.json`

```json
{
  "name": "@my-org/my-feature",
  "version": "1.0.0",
  "license": "EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0",
  "theiaExtensions": [{
    "frontend": "lib/browser/my-feature-frontend-module",
    "backend":  "lib/node/my-feature-backend-module"
  }],
  "dependencies": {
    "@theia/core": "^1.71.0"
  },
  "devDependencies": {
    "@theia/core": "^1.71.0"
  }
}
```

### Minimal `tsconfig.json`

```json
{
  "extends": "../../configs/base.tsconfig.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "lib"
  }
}
```

### SPDX Copyright Header (required on every new file)

```ts
// *****************************************************************************
// Copyright (C) 2025 Your Company Name.
//
// This program and the accompanying materials are made available under the
// terms of the Eclipse Public License v. 2.0 which is available at
// http://www.eclipse.org/legal/epl-2.0.
//
// SPDX-License-Identifier: EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0
// *****************************************************************************
```

---

## 2. Registering Commands

```ts
// src/browser/my-feature-contribution.ts
import { CommandContribution, CommandRegistry } from '@theia/core';
import { injectable } from '@theia/core/shared/inversify';
import { Command } from '@theia/core';

export const MY_COMMAND: Command = {
    id: 'my-feature:doSomething',
    label: 'My Feature: Do Something'
};

@injectable()
export class MyFeatureCommandContribution implements CommandContribution {
    registerCommands(registry: CommandRegistry): void {
        registry.registerCommand(MY_COMMAND, {
            execute: () => this.doSomething()
        });
    }

    protected doSomething(): void {
        console.log('Hello from my feature!');
    }
}
```

```ts
// src/browser/my-feature-frontend-module.ts
import { CommandContribution } from '@theia/core';
import { ContainerModule } from '@theia/core/shared/inversify';
import { MyFeatureCommandContribution } from './my-feature-contribution';

export default new ContainerModule(bind => {
    bind(MyFeatureCommandContribution).toSelf().inSingletonScope();
    bind(CommandContribution).toService(MyFeatureCommandContribution);
});
```

---

## 3. Adding Menu Items

```ts
import { MenuContribution, MenuModelRegistry, CommonMenus } from '@theia/core';
import { injectable } from '@theia/core/shared/inversify';
import { MY_COMMAND } from './my-feature-contribution';

@injectable()
export class MyFeatureMenuContribution implements MenuContribution {
    registerMenus(menus: MenuModelRegistry): void {
        menus.registerMenuAction(CommonMenus.EDIT_FIND, {
            commandId: MY_COMMAND.id,
            label: 'My Feature: Do Something'
        });
    }
}
```

---

## 4. Creating a Widget

```ts
// src/browser/my-widget.tsx
import { ReactWidget } from '@theia/core/lib/browser';
import { injectable } from '@theia/core/shared/inversify';
import * as React from '@theia/core/shared/react';

@injectable()
export class MyWidget extends ReactWidget {
    static readonly ID = 'my-feature-widget';
    static readonly LABEL = 'My Feature';

    constructor() {
        super();
        this.id = MyWidget.ID;
        this.title.label = MyWidget.LABEL;
        this.title.closable = true;
    }

    protected render(): React.ReactNode {
        return <div>Hello from My Feature Widget!</div>;
    }
}
```

Bind and open:

```ts
// In frontend module:
import { WidgetFactory, bindViewContribution, FrontendApplicationContribution } from '@theia/core/lib/browser';
import { MyWidget } from './my-widget';
import { MyViewContribution } from './my-view-contribution';

bindViewContribution(bind, MyViewContribution);
bind(FrontendApplicationContribution).toService(MyViewContribution);
bind(MyWidget).toSelf();
bind(WidgetFactory).toDynamicValue(ctx => ({
    id: MyWidget.ID,
    createWidget: () => ctx.container.get(MyWidget)
})).inSingletonScope();
```

---

## 5. Overriding a Core Service (Without Forking)

Theia allows you to override any singleton by rebinding it in your module:

```ts
// Override the default label provider contribution:
import { LabelProviderContribution } from '@theia/core/lib/browser';
import { MyCustomLabelProvider } from './my-custom-label-provider';

// In your ContainerModule:
bind(MyCustomLabelProvider).toSelf().inSingletonScope();
bind(LabelProviderContribution).toService(MyCustomLabelProvider);
// The contribution provider will now include yours alongside the defaults.
```

For full replacement of a singleton service:

```ts
// Rebind completely:
bind(OriginalService).toConstantValue(new MyReplacementService());
// OR
rebind(OriginalService).to(MyReplacementService).inSingletonScope();
```

---

## 6. Adding Preferences

```ts
// src/common/my-feature-preferences.ts
import { PreferenceSchema } from '@theia/core/lib/common/preferences/preference-schema';

export const MY_FEATURE_PREFERENCES_SCHEMA: PreferenceSchema = {
    type: 'object',
    properties: {
        'myFeature.enabled': {
            type: 'boolean',
            default: true,
            description: 'Enable My Feature functionality'
        },
        'myFeature.maxItems': {
            type: 'number',
            default: 50,
            minimum: 1,
            maximum: 200,
            description: 'Maximum number of items to display'
        }
    }
};

// In frontend module:
// import { bindPreferenceProvider } from '@theia/core/lib/browser/preferences';
// bindPreferenceProvider(bind, MyFeaturePreferences);
```

---

## 7. Frontend ↔ Backend Services

For services that need Node.js backend access:

```ts
// src/common/my-service.ts
export const MyServicePath = '/services/my-service';
export const MyService = Symbol('MyService');
export interface MyService {
    getData(): Promise<string[]>;
}
```

```ts
// src/node/my-service-impl.ts
import { injectable } from '@theia/core/shared/inversify';
import { MyService } from '../common/my-service';

@injectable()
export class MyServiceImpl implements MyService {
    async getData(): Promise<string[]> {
        return ['item1', 'item2'];
    }
}
```

```ts
// src/node/my-feature-backend-module.ts
import { ConnectionHandler, JsonRpcConnectionHandler } from '@theia/core';
import { ContainerModule } from '@theia/core/shared/inversify';
import { MyService, MyServicePath } from '../common/my-service';
import { MyServiceImpl } from './my-service-impl';

export default new ContainerModule(bind => {
    bind(MyServiceImpl).toSelf().inSingletonScope();
    bind(MyService).toService(MyServiceImpl);
    bind(ConnectionHandler).toDynamicValue(ctx =>
        new JsonRpcConnectionHandler(MyServicePath, () => ctx.container.get(MyService))
    ).inSingletonScope();
});
```

```ts
// src/browser/my-feature-frontend-module.ts (add to existing)
import { ServiceConnectionProvider, RemoteConnectionProvider } from '@theia/core/lib/browser/messaging';
import { MyService, MyServicePath } from '../common/my-service';

bind(MyService).toDynamicValue(ctx => {
    const provider = ctx.container.get<ServiceConnectionProvider>(RemoteConnectionProvider);
    return provider.createProxy<MyService>(MyServicePath);
}).inSingletonScope();
```

---

## 8. Internationalization (i18n)

```ts
import { nls } from '@theia/core';

// For strings that exist in VS Code's language packs:
const label = nls.localizeByDefault('Close');

// For custom Theia strings:
const label = nls.localize('theia/my-feature/myKey', 'My Default Text');

// With dynamic args (never interpolate!):
const msg = nls.localize('theia/my-feature/hello', 'Hello {0}!', userName);

// Commands use helper functions:
import { Command } from '@theia/core';
const MY_CMD = Command.toLocalizedCommand(
    { id: 'my-feature:action', label: 'My Action' },
    'theia/my-feature/myAction'
);
```

---

## 9. Testing Your Extension

```ts
// src/browser/my-feature-contribution.spec.ts
import { expect } from 'chai';
import { Container } from '@theia/core/shared/inversify';
import { MyFeatureCommandContribution } from './my-feature-contribution';

describe('MyFeatureCommandContribution', () => {
    let contribution: MyFeatureCommandContribution;

    beforeEach(() => {
        const container = new Container();
        container.bind(MyFeatureCommandContribution).toSelf();
        contribution = container.get(MyFeatureCommandContribution);
    });

    it('should register commands', () => {
        // test implementation
        expect(contribution).to.be.instanceOf(MyFeatureCommandContribution);
    });
});
```

Run tests:

```bash
# All tests
npm run test

# Single package
npx lerna run test --scope @my-org/my-feature

# Single file
npx mocha ./packages/my-feature/lib/browser/my-feature-contribution.spec.js
```

---

## 10. Checklist Before Shipping a PR

- [ ] SPDX copyright header on every new file
- [ ] No imports crossing platform boundaries (browser ↔ node)
- [ ] All singletons bound with `inSingletonScope()`
- [ ] `bindRootContributionProvider` used (not `bindContributionProvider`) in top-level modules
- [ ] No `null` — use `undefined`
- [ ] All user-facing strings localized with `nls.localize`
- [ ] No inline arrow functions or `.bind(this)` in JSX
- [ ] Explicit return types on all public methods
- [ ] Tests exist for new behavior
- [ ] No raw OS paths — use URI strings
