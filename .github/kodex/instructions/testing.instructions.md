---
applyTo: "**/*.spec.ts,**/*.ui-spec.ts,**/*.slow-spec.ts"
description: "Testing conventions for Theia extension test files"
---

# Theia Testing Instructions

<!-- Applies to all test files in the project.
     Pairs with extension-coding.instructions.md — all style rules still apply. -->

## Test File Types and Naming

| Type             | Suffix          | Runs in CI | Purpose                        |
| ---------------- | --------------- | ---------- | ------------------------------ |
| Unit test        | `.spec.ts`      | Yes        | Logic, services, contributions |
| UI test          | `.ui-spec.ts`   | Separately | Playwright-based browser tests |
| Slow/integration | `.slow-spec.ts` | Separately | Network, file system, long ops |

**Rule:** A test file lives next to the file it tests.
`src/browser/my-service.ts` → `src/browser/my-service.spec.ts`

Test helpers and mocks go in `src/<platform>/test/`:

- `src/browser/test/mock-my-service.ts`
- `src/node/test/test-helper.ts`

Test resources (config files, scripts) go in `test-resources/`.

## Basic Unit Test Structure

```ts
// src/browser/my-service.spec.ts
import { expect } from "chai";
import { Container } from "@theia/core/shared/inversify";
import { MyService } from "./my-service";
import { MockDependency } from "./test/mock-dependency";

describe("MyService", () => {
    let service: MyService;
    let mockDep: MockDependency;

    beforeEach(() => {
        const container = new Container();
        mockDep = new MockDependency();
        container.bind(MyService).toSelf();
        container.bind(Dependency).toConstantValue(mockDep);
        service = container.get(MyService);
    });

    it("should return a value when called", () => {
        const result = service.compute("input");
        expect(result).to.equal("expected");
    });

    it("should handle empty input gracefully", () => {
        const result = service.compute("");
        expect(result).to.be.undefined;
    });
});
```

## Testing DI Contributions

```ts
import { CommandRegistry } from "@theia/core";
import { MyCommandContribution } from "./my-command-contribution";

describe("MyCommandContribution", () => {
    let contribution: MyCommandContribution;
    let registry: CommandRegistry;

    beforeEach(() => {
        const container = new Container();
        container.bind(MyCommandContribution).toSelf();
        container.bind(CommandRegistry).toSelf().inSingletonScope();
        contribution = container.get(MyCommandContribution);
        registry = container.get(CommandRegistry);
    });

    it("should register MY_COMMAND", () => {
        contribution.registerCommands(registry);
        expect(registry.getCommand("my-feature:doSomething")).to.exist;
    });
});
```

## Testing AI Agents

```ts
import { MyAgent } from "./my-agent";
import { MockPromptService } from "@theia/ai-core/lib/browser/test/mock-prompt-service";

describe("MyAgent", () => {
    let agent: MyAgent;

    beforeEach(() => {
        const container = new Container();
        container.bind(MyAgent).toSelf();
        container.bind(PromptService).toConstantValue(new MockPromptService());
        agent = container.get(MyAgent);
    });

    it("should have correct id", () => {
        expect(agent.id).to.equal("MyAgent");
    });

    it("should declare language model requirements", () => {
        expect(agent.languageModelRequirements).to.have.length.greaterThan(0);
    });
});
```

## Testing Async Code

```ts
it("should resolve data asynchronously", async () => {
    const result = await service.fetchData("query");
    expect(result).to.be.an("array").with.length.greaterThan(0);
});

it("should reject on invalid input", async () => {
    await expect(service.fetchData("")).to.be.rejectedWith(Error);
});
```

## Running Tests

```bash
# All tests
npm run test

# Single package
npx lerna run test --scope @theia/my-package

# Single compiled file (after compile)
npx mocha ./packages/my-feature/lib/browser/my-service.spec.js

# Watch mode (add "test:watch": "theiaext test:watch" to package.json)
npx lerna run test:watch --scope @theia/my-package
```

## Test Coverage

Coverage is collected automatically by NYC (Istanbul) during `npm run test`.
Reports appear in `coverage/` after a test run.

## Common Mistakes

| Mistake                              | Fix                                                      |
| ------------------------------------ | -------------------------------------------------------- |
| `.spec.ts` without compiling first   | Run `npm run compile` or use watch mode                  |
| Importing from `lib/` in test source | Import from `src/` in spec files                         |
| Using `new MyService()` directly     | Use InversifyJS Container to get proper DI               |
| Shared mutable state between tests   | Use `beforeEach` to reset state                          |
| Slow test in `.spec.ts`              | Move to `.slow-spec.ts`                                  |
| DOM-dependent test without JSDOM     | Use `@theia/private-test-setup` or move to `.ui-spec.ts` |
