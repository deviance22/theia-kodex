# Skill: Customize IDE Look & Feel

> **STOP — Planning Gate**
> This skill requires a completed, user-confirmed Feature Design Document (FDD) before any files are created.
> If you do not have one, run `skills/feature-planning.md` first.
> Load `agents/theia-planner.agent.md` to conduct the planning session.

**When to use:** After a confirmed FDD — a developer is changing the visual appearance of the IDE (brand colors, fonts, themes, tab styles, status bar, icon sets, application name).

**Reference:** `knowledge/06-look-and-feel.md` for full technical detail.

---

## Step 0: Determine the Scope

Ask the developer:
1. **Brand/name change only?** → Level 1 (config) is enough.
2. **Need a standard VS Code theme?** → Level 2 (plugin) is enough.
3. **Custom font, tab size, scrollbar?** → Level 3 (CSS override).
4. **New color tokens that switch with dark/light mode?** → Level 4 (`ColorContribution`).
5. **Widget styling that differs between dark/light/high-contrast?** → Level 5 (`StylingParticipant`).
6. **Complete custom color palette from scratch?** → Level 6 (VS Code theme extension).

Start at the lowest applicable level. Do not write TypeScript if CSS config suffices.

---

## Level 1 — Application Name & Default Theme (5 minutes)

Edit the app's `package.json` (e.g., `examples/browser/package.json`):

```json
"theia": {
  "frontend": {
    "config": {
      "applicationName": "Your IDE Name",
      "defaultTheme": { "light": "light", "dark": "dark" },
      "preferences": {
        "workbench.colorTheme": "Default Dark+",
        "workbench.iconTheme": "vs-seti",
        "editor.fontFamily": "'JetBrains Mono', Menlo, monospace",
        "editor.fontSize": 14
      }
    }
  }
}
```

Rebuild: `npm run build:browser`

---

## Level 2 — Install a VS Code Theme (10–15 minutes)

**Option A: From Open VSX (auto-download)**

Add to the app's `package.json`:

```json
"theiaPlugins": {
  "one-dark-pro": "https://open-vsx.org/api/zhuangtongfa/material-theme/3.17.7/file/zhuangtongfa.material-theme-3.17.7.vsix"
}
```

Run `npm run download:plugins`, then set as default via preferences.

**Option B: Drop a `.vsix` manually**

Copy the `.vsix` file into the `plugins/` directory at the repo root. Set theme ID in preferences.

---

## Level 3 — CSS Variable Overrides (30–60 minutes)

### 1. Create the CSS file

Create `src/browser/style/brand.css` in your extension package:

```css
:root {
    /* Font */
    --theia-ui-font-family: 'Inter', 'Segoe UI', Arial, sans-serif;

    /* Tab height */
    --theia-private-horizontal-tab-height: 38px;
    --theia-horizontal-toolbar-height: var(--theia-private-horizontal-tab-height);

    /* Scrollbar */
    --theia-scrollbar-width: 8px;
    --theia-scrollbar-rail-width: 8px;

    /* Padding */
    --theia-ui-padding: 5px;
}
```

### 2. Import in your frontend module

```ts
// At the top of src/browser/your-frontend-module.ts
import './style/brand.css';
```

### 3. Rebuild and verify

```bash
npm run build:browser
npm run start:browser
```

Open DevTools → Elements → inspect `:root` to verify the variables are overriding correctly.

---

## Level 4 — Color Contribution (1–2 hours)

Use when you need **named brand colors that automatically switch between dark/light themes**.

### 1. Create the contribution class

```ts
// src/browser/brand-color-contribution.ts
// *****************************************************************************
// Copyright (C) 2025 <Your Company>.
// SPDX-License-Identifier: EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0
// *****************************************************************************
import { injectable } from '@theia/core/shared/inversify';
import { ColorContribution } from '@theia/core/lib/browser/color-application-contribution';
import { ColorRegistry } from '@theia/core/lib/browser/color-registry';

@injectable()
export class BrandColorContribution implements ColorContribution {
    registerColors(colors: ColorRegistry): void {
        colors.register(
            {
                id: 'brand.accent',
                defaults: { dark: '#E87B1E', light: '#C25A00', hcDark: '#FF8C00', hcLight: '#8B3A00' },
                description: 'Brand accent color'
            },
            {
                id: 'brand.sidebarHeader',
                defaults: { dark: '#1A1A2E', light: '#F0F4FF', hcDark: '#000000', hcLight: '#FFFFFF' },
                description: 'Brand sidebar header background'
            }
        );
    }
}
```

### 2. Bind in the frontend module

```ts
import { ColorContribution } from '@theia/core/lib/browser/color-application-contribution';
import { BrandColorContribution } from './brand-color-contribution';

// In ContainerModule:
bind(BrandColorContribution).toSelf().inSingletonScope();
bind(ColorContribution).toService(BrandColorContribution);
```

### 3. Use in CSS

```css
.my-sidebar-header {
    background-color: var(--theia-brand-sidebarHeader);
    border-bottom: 2px solid var(--theia-brand-accent);
}
```

Note: `.` in color IDs become `-` in CSS variable names.

---

## Level 5 — Styling Participant (2–4 hours)

Use when CSS variables alone cannot capture the difference between dark/light/high-contrast.

```ts
// src/browser/brand-styling-participant.ts
// *****************************************************************************
// Copyright (C) 2025 <Your Company>.
// SPDX-License-Identifier: EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0
// *****************************************************************************
import { injectable } from '@theia/core/shared/inversify';
import { StylingParticipant, ColorTheme, CssStyleCollector } from '@theia/core/lib/browser/styling-service';
import { isHighContrast } from '@theia/core/lib/common/theme';

@injectable()
export class BrandStylingParticipant implements StylingParticipant {
    registerThemeStyle(theme: ColorTheme, collector: CssStyleCollector): void {
        const accent = theme.getColor('brand.accent');

        if (accent) {
            collector.addRule(`
                .brand-header {
                    border-bottom: 2px solid ${accent};
                }
            `);
        }

        if (isHighContrast(theme.type)) {
            collector.addRule(`
                .brand-header {
                    border: 2px solid ButtonText;
                }
            `);
        }
    }
}
```

Bind:

```ts
import { StylingParticipant } from '@theia/core/lib/browser/styling-service';
bind(BrandStylingParticipant).toSelf().inSingletonScope();
bind(StylingParticipant).toService(BrandStylingParticipant);
```

---

## Level 6 — Custom VS Code Theme Extension (half-day to 1 day)

### File structure

```
my-theme-extension/
├── package.json        ← VS Code extension manifest
└── themes/
    ├── my-dark.json
    └── my-light.json
```

### `package.json`

```json
{
  "name": "my-ide-theme",
  "version": "1.0.0",
  "publisher": "my-org",
  "engines": { "vscode": "^1.80.0" },
  "contributes": {
    "themes": [
      { "label": "My Dark", "uiTheme": "vs-dark", "path": "./themes/my-dark.json" },
      { "label": "My Light", "uiTheme": "vs",     "path": "./themes/my-light.json" }
    ]
  }
}
```

### `themes/my-dark.json` (minimal)

```json
{
  "name": "My Dark",
  "type": "dark",
  "colors": {
    "editor.background": "#1A1A2E",
    "editor.foreground": "#E0E0E0",
    "activityBar.background": "#0F0F23",
    "activityBar.foreground": "#FFFFFF",
    "sideBar.background": "#16213E",
    "sideBar.foreground": "#CCCCCC",
    "statusBar.background": "#E87B1E",
    "statusBar.foreground": "#FFFFFF",
    "tab.activeBackground": "#1A1A2E",
    "tab.inactiveBackground": "#0F0F23",
    "tab.activeForeground": "#FFFFFF",
    "tab.inactiveForeground": "#AAAAAA"
  },
  "tokenColors": []
}
```

### Package and deploy

```bash
# Install vsce if needed
npm install -g @vscode/vsce

# Package the extension
cd my-theme-extension
vsce package

# Drop the .vsix in the plugins/ directory
cp my-ide-theme-1.0.0.vsix ../../plugins/

# Set as default in app's package.json preferences:
# "workbench.colorTheme": "My Dark"
```

---

## Accessibility Checklist

After any color change, verify:

- [ ] Text contrast ≥ 4.5:1 (normal), ≥ 3:1 (large text, icons) — check with <https://webaim.org/resources/contrastchecker/>
- [ ] No information conveyed by color alone (pair with icon or text)
- [ ] Icons use `currentColor` in SVG fill/stroke
- [ ] Tested in `hc-black` and `hc-light` themes
- [ ] No inline styles in any component (use CSS variables)
- [ ] No hard-coded hex values in TypeScript — route through `ColorContribution`

---

## Debugging Tips

```bash
# Inspect CSS variables live
# Open DevTools (F12) → Console:
getComputedStyle(document.documentElement).getPropertyValue('--theia-editor-background')

# See all --theia-* variables:
[...document.styleSheets]
  .flatMap(s => [...s.cssRules])
  .filter(r => r.selectorText === ':root')
  .flatMap(r => r.style.cssText.split(';'))
  .filter(s => s.trim().startsWith('--theia'))
```
