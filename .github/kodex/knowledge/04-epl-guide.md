# 04 — EPL-2.0 License Guide for Product Development

> Plain-language guidance for engineers building a commercial product on Theia.
> **This is not legal advice.** Consult your legal team for binding decisions.

---

## What License is Theia?

Theia is dual-licensed under:
- **EPL-2.0** (Eclipse Public License 2.0)
- **GPL-2.0-only WITH Classpath-exception-2.0**

The Classpath Exception is the key — it explicitly allows you to link Theia into your product
**without the GPL copyleft propagating to your proprietary code**.

---

## What You Can Do

| Action | Allowed? |
|---|---|
| Use Theia as a dependency in a commercial product | ✅ Yes |
| Distribute a compiled app that includes Theia | ✅ Yes (with attribution) |
| Write closed-source extensions that run on Theia | ✅ Yes |
| Modify Theia source for internal use | ✅ Yes |
| Distribute a modified version of Theia | ✅ Yes, but modified files must stay EPL-2.0 |
| Copy Theia source verbatim into your proprietary codebase | ⚠️ Legal review required |

---

## Required Obligations When Distributing

1. **Include a copy of the EPL-2.0 license** in your distribution.
2. **Provide access to Theia's source** (or a written offer to do so) for any EPL-covered code you distribute in binary form.
3. **Retain copyright notices** from original Theia files.
4. If you **modify a Theia source file**, the modified file must remain under EPL-2.0.

---

## The "Extension" Safe Zone

Because Theia is designed as a framework and because of the Classpath Exception:

- Code you write in **separate npm packages** that are loaded by Theia at runtime is considered "separate works".
- Your extension packages can use **any license** (including proprietary/closed-source).
- This is the same model used by VS Code extensions, Eclipse plug-ins, and IntelliJ plugins.

**Bottom line:** Keep your product code in packages outside the `packages/` directory. Do not modify files inside `packages/`.

---

## File Header Requirements

Every file you create **inside this repository** (under EPL-2.0) must have:

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

For files in **your product repository** (outside this repo), use whatever header matches your product's license.

---

## When You Must Touch Core Theia Files

Sometimes a bug fix or missing extension point requires modifying a core file.
In that case:

1. **Prefer upstream first** — file a GitHub issue or PR against `eclipse-theia/theia`. This is the safest long-term path.
2. **If you must patch locally** — the patched file remains EPL-2.0. Document the patch clearly.
3. **Use `theia-patch`** — the repo includes a patching mechanism via the `theia-patch` script run during `npm install`. This keeps patches trackable and reversible.
4. When Theia updates, your patches must be reapplied and may need updating.

---

## Attribution

In your product's About dialog or documentation, include something like:

> This product includes software developed by the Eclipse Theia project.
> Source: <https://github.com/eclipse-theia/theia>
> License: EPL-2.0 OR GPL-2.0-only WITH Classpath-exception-2.0

---

## Third-Party Dependencies

Theia uses many third-party npm packages. The CI `license-check` job validates these.
When you add new dependencies to your product:

- Run the `license-check-config.json` validation
- Ensure new packages don't introduce GPL (without Classpath Exception) dependencies
- Avoid AGPL, SSPL, or Commons Clause licenses without legal review
