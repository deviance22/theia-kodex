# 03 — Commands Cheat Sheet

> Copy-paste reference for daily development workflows.

---

## Setup

```bash
# Install all dependencies (run from repo root)
npm install
```

---

## Build

```bash
# Compile TypeScript only (fast — use during development)
npm run compile

# Build everything + bundle the browser app (required for UI testing)
npm run build:browser

# Build a single package (TypeScript only)
npx lerna run compile --scope @theia/package-name

# Watch mode — recompile on change (browser + electron)
npm run watch

# Watch a single package and its dependencies
npx lerna run watch --scope @theia/package-name --include-filtered-dependencies --parallel
```

---

## Run

```bash
# Start browser app at http://localhost:3000
npm run start:browser

# Start Electron desktop app
npm run start:electron
```

---

## Test

```bash
# Run all tests
npm run test

# Run tests for a single package
npx lerna run test --scope @theia/package-name

# Run a single compiled test file
npx mocha ./packages/my-feature/lib/browser/my-service.spec.js

# Enable watch mode for a package's tests
# (add "test:watch": "theiaext test:watch" to the package's package.json first)
npx lerna run test:watch --scope @theia/package-name
```

---

## Lint

```bash
# Lint all packages
npm run lint

# Lint and auto-fix
npm run lint:fix
```

---

## Useful Lerna Commands

```bash
# List all packages
npx lerna list

# Build packages in dependency order
npx lerna run build

# Run any npm script across all packages
npx lerna run <script>

# Run on packages that changed since main branch
npx lerna run compile --since origin/main
```

---

## Generating a New Package (Manual)

No CLI generator exists — copy an existing simple package:

```bash
# From repo root — copy a simple package as a template
cp -r packages/getting-started packages/my-feature

# Then update:
# 1. packages/my-feature/package.json  → name, description, dependencies
# 2. packages/my-feature/tsconfig.json → (usually no changes needed)
# 3. src/ → replace with your code
# 4. Add to your app's package.json dependencies
```

---

## TypeScript Project References

The repo uses TypeScript project references for fast incremental builds.
After adding a new package, register it:

1. Add to `configs/base.tsconfig.json` references array (if it's a dependency of others)
2. Run `npm install` — the `compute-references` hook regenerates reference graphs

---

## Environment Variables (AI features)

```bash
# OpenAI
OPENAI_API_KEY=sk-...

# Anthropic
ANTHROPIC_API_KEY=sk-ant-...

# Ollama (local, default endpoint)
# No env var needed — configure via preferences in the running app
```

---

## Checking What's Installed

```bash
# Node version (must be ≥ 20)
node --version

# Package versions
cat lerna.json | grep version

# All Theia packages
npx lerna list --all
```
