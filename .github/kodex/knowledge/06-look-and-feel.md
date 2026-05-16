# 06 — Look & Feel Customization

> Everything a developer needs to change the visual appearance of the IDE: application name, colors, themes, fonts, layout styling, and icon sets.
> All approaches work **without modifying core Theia files**.

---

## Theming Layers (Simplest → Most Powerful)

```
Level 1: Application Config         → app name, default theme, default preferences
Level 2: VS Code theme plugin       → full color scheme via .vsix
Level 3: CSS variable overrides     → tweak any --theia-* variable in a CSS file
Level 4: ColorContribution          → register new semantic color tokens (TypeScript)
Level 5: StylingParticipant         → dynamic CSS rules that react to theme type
Level 6: Custom VS Code theme       → author a full color theme extension
```

Start at Level 1 and only go deeper if you need more control.

---

## Level 1 — Application Config (`package.json`)

The fastest way to brand the IDE is inside the app's `package.json`
(e.g., `examples/browser/package.json` or your product app's `package.json`):

```json
{
  "theia": {
    "frontend": {
      "config": {
        "applicationName": "Kodex IDE",
        "defaultTheme": {
          "light": "light",
          "dark": "dark"
        },
        "preferences": {
          "workbench.colorTheme": "Default Dark+",
          "workbench.iconTheme": "vs-seti",
          "editor.fontFamily": "'JetBrains Mono', Menlo, monospace",
          "editor.fontSize": 14
        }
      }
    }
  }
}
```

`defaultTheme` can be:
- `"dark"` / `"light"` — Theia built-ins
- `{ "light": "light-plus", "dark": "dark-plus" }` — adapt to OS theme
- Any VS Code theme ID installed as a plugin

Changes here take effect after `npm run build:browser`.

---

## Level 2 — Install a VS Code Theme Plugin

VS Code themes ship as `.vsix` files and work in Theia without any code changes.

### Option A: Drop a `.vsix` in the plugins directory

1. Download the theme `.vsix` (e.g., from `open-vsx.org`)
2. Place it in the `plugins/` directory at the repo root
3. Set it as default in `package.json`:

```json
"preferences": {
  "workbench.colorTheme": "One Dark Pro"
}
```

### Option B: Configure in `package.json` (auto-download)

```json
"theiaPlugins": {
  "vscode-theme-one-dark": "https://open-vsx.org/api/zhuangtongfa/material-theme/3.17.7/file/zhuangtongfa.material-theme-3.17.7.vsix"
}
```

Then run `npm run download:plugins` (or the equivalent in your app).

### Available built-in themes (no plugin needed)

| Theme ID | Type |
|---|---|
| `dark` | Dark |
| `light` | Light |
| `hc-black` | High Contrast Dark |
| `hc-light` | High Contrast Light |

---

## Level 3 — CSS Variable Overrides

Theia exposes every visual dimension as `--theia-*` CSS custom properties.
Overriding them in your own CSS file requires **no TypeScript** and **no core changes**.

### Step 1 — Create the CSS file

```css
/* src/browser/style/my-brand.css */

:root {
    /* Override the base font */
    --theia-ui-font-family: 'Inter', 'Segoe UI', sans-serif;
    --theia-ui-font-size1: 13px;

    /* Override tab height */
    --theia-private-horizontal-tab-height: 40px;
    --theia-horizontal-toolbar-height: var(--theia-private-horizontal-tab-height);

    /* Scrollbar width */
    --theia-scrollbar-width: 8px;
    --theia-scrollbar-rail-width: 8px;
}

/* Custom tab styling */
.lm-TabBar .lm-TabBar-tab.lm-mod-current {
    border-bottom: 2px solid var(--theia-focusBorder);
}

/* Custom status bar */
#theia-statusBar {
    font-weight: 600;
}
```

### Step 2 — Load the CSS from your frontend module

```ts
// src/browser/my-brand-frontend-module.ts
import '../../src/browser/style/my-brand.css';
```

Or for webpack-style loading (standard pattern):

```ts
// At the top of your frontend module file:
import './style/my-brand.css';
```

Make sure your `package.json` lists the CSS in the correct location after compilation,
and that webpack (used during `npm run build:browser`) picks it up via the module import.

### Key `--theia-*` Variables Reference

**Typography:**

```css
--theia-ui-font-family         /* UI chrome font */
--theia-ui-font-size1          /* Base size (13px default) */
--theia-code-font-family       /* Editor/terminal monospace font */
--theia-code-font-size         /* Editor font size */
--theia-content-line-height    /* Line height for content areas */
```

**Colors (set by themes, override with caution):**

```css
--theia-editor-background           /* Editor background */
--theia-sideBar-background          /* Sidebar background */
--theia-activityBar-background      /* Activity bar (leftmost strip) */
--theia-statusBar-background        /* Status bar */
--theia-tab-activeBackground        /* Active tab */
--theia-tab-inactiveBackground      /* Inactive tab */
--theia-focusBorder                 /* Focus ring color */
--theia-foreground                  /* General text color */
```

**Layout:**

```css
--theia-border-width                         /* 1px default */
--theia-private-horizontal-tab-height        /* Tab bar height (35px) */
--theia-scrollbar-width                      /* Scrollbar width */
--theia-ui-padding                           /* General padding unit */
--theia-icon-size                            /* Icon size (16px) */
```

---

## Level 4 — `ColorContribution` (Register Custom Color Tokens)

Use this when you need color tokens that integrate with the theme-switching system
(the token changes value automatically when the user switches between dark and light).

```ts
// src/browser/my-brand-colors.ts
import { injectable } from '@theia/core/shared/inversify';
import { ColorContribution } from '@theia/core/lib/browser/color-application-contribution';
import { ColorRegistry } from '@theia/core/lib/browser/color-registry';

@injectable()
export class MyBrandColorContribution implements ColorContribution {
    registerColors(colors: ColorRegistry): void {
        colors.register(
            {
                id: 'myBrand.accentColor',
                defaults: {
                    dark: '#E87B1E',
                    light: '#C25A00',
                    hcDark: '#FF8C00',
                    hcLight: '#8B3A00'
                },
                description: 'My brand accent color'
            },
            {
                id: 'myBrand.headerBackground',
                defaults: {
                    dark: '#1A1A2E',
                    light: '#F0F4FF',
                    hcDark: '#000000',
                    hcLight: '#FFFFFF'
                },
                description: 'My brand header background'
            }
        );
    }
}
```

Bind it:

```ts
// In your ContainerModule:
import { ColorContribution } from '@theia/core/lib/browser/color-application-contribution';

bind(MyBrandColorContribution).toSelf().inSingletonScope();
bind(ColorContribution).toService(MyBrandColorContribution);
```

After registration, use in CSS as `var(--theia-myBrand-accentColor)`.
The `.` in the color ID is automatically converted to `-` in the CSS variable name.

---

## Level 5 — `StylingParticipant` (Dynamic Theme-Reactive CSS)

Use when you need CSS rules that differ between dark, light, and high-contrast modes.
The participant runs every time the theme changes.

```ts
// src/browser/my-brand-styling.ts
import { injectable } from '@theia/core/shared/inversify';
import { StylingParticipant, ColorTheme, CssStyleCollector } from '@theia/core/lib/browser/styling-service';
import { isHighContrast } from '@theia/core/lib/common/theme';

@injectable()
export class MyBrandStylingParticipant implements StylingParticipant {
    registerThemeStyle(theme: ColorTheme, collector: CssStyleCollector): void {
        const accent = theme.getColor('myBrand.accentColor');
        const isDark = theme.type === 'dark';

        if (accent) {
            collector.addRule(`
                .my-brand-header {
                    border-bottom: 2px solid ${accent};
                }
                .my-brand-button {
                    background-color: ${accent};
                    color: ${isDark ? '#ffffff' : '#000000'};
                }
            `);
        }

        if (isHighContrast(theme.type)) {
            collector.addRule(`
                .my-brand-header {
                    border-bottom: 3px solid ButtonText;
                }
            `);
        }
    }
}
```

Bind it:

```ts
import { StylingParticipant } from '@theia/core/lib/browser/styling-service';

bind(MyBrandStylingParticipant).toSelf().inSingletonScope();
bind(StylingParticipant).toService(MyBrandStylingParticipant);
```

---

## Level 6 — Custom VS Code Color Theme Extension

For complete control over all VS Code-compatible color tokens, author a theme extension.

### Minimal theme extension structure

```
my-theme/
├── package.json
└── themes/
    └── my-dark-theme.json
```

**`package.json`:**

```json
{
  "name": "my-ide-theme",
  "version": "1.0.0",
  "publisher": "my-org",
  "engines": { "vscode": "^1.80.0" },
  "contributes": {
    "themes": [{
      "label": "My Dark Theme",
      "uiTheme": "vs-dark",
      "path": "./themes/my-dark-theme.json"
    }]
  }
}
```

**`themes/my-dark-theme.json`:**

```json
{
  "name": "My Dark Theme",
  "type": "dark",
  "colors": {
    "editor.background": "#1A1A2E",
    "editor.foreground": "#E0E0E0",
    "activityBar.background": "#0F0F23",
    "sideBar.background": "#16213E",
    "statusBar.background": "#E87B1E",
    "statusBar.foreground": "#FFFFFF",
    "tab.activeBackground": "#1A1A2E",
    "tab.inactiveBackground": "#0F0F23"
  },
  "tokenColors": [
    {
      "scope": ["keyword"],
      "settings": { "foreground": "#C678DD" }
    },
    {
      "scope": ["string"],
      "settings": { "foreground": "#98C379" }
    }
  ]
}
```

Package as `.vsix` and deploy to your `plugins/` directory.

---

## Accessibility Requirements (WCAG 2.2 AA)

When customizing colors, always verify:

- **Contrast ratio ≥ 4.5:1** for normal text; ≥ 3:1 for large text and UI components
- **Never use color alone** to convey information — pair with text, icons, or patterns
- **Icons must use `currentColor`** so they adapt in forced-colors / high-contrast mode
- **Test all theme types**: dark, light, `hc-black`, `hc-light`

Tools: <https://webaim.org/resources/contrastchecker/>

---

## Icon Themes

Icon themes are VS Code extensions (`"contributes": { "iconThemes": [...] }`).
Set the default via preference:

```json
"preferences": {
  "workbench.iconTheme": "vs-seti"
}
```

Popular options available on Open VSX: `vscode-icons`, `material-icon-theme`, `vs-seti`.

---

## Common Scenarios

| Goal | Approach |
|---|---|
| Change the IDE name in the title bar | Level 1 — `applicationName` in `package.json` |
| Change the default theme | Level 1 — `defaultTheme` in `package.json` |
| Use a popular VS Code theme (One Dark, etc.) | Level 2 — install theme plugin |
| Change the font | Level 3 — CSS variable `--theia-ui-font-family` |
| Change tab height or scrollbar size | Level 3 — CSS variable override |
| Add a branded accent color that adapts to themes | Level 4 — `ColorContribution` |
| Style a custom widget differently in dark vs. light | Level 5 — `StylingParticipant` |
| Full custom color palette from scratch | Level 6 — custom VS Code theme extension |
