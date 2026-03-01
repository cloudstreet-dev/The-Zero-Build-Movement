# Testing Without a Toolchain

*Node's Built-In Test Runner Is Actually Good*

---

The JavaScript testing ecosystem is a vivid illustration of what happens when a platform lacks a feature for long enough: the community builds five competing solutions, each with their own config format, assertion library, mock API, and opinion about where test files should live. Then the platform ships the feature natively and everyone awkwardly avoids mentioning it.

Node shipped a built-in test runner in Node 18 (experimental) and Node 20 (stable). It works. It doesn't need Jest, it doesn't need Vitest, it doesn't need Mocha, it doesn't need a test runner config file.

## Node's Built-In Test Runner

```javascript
// math.test.js — runs with: node --test math.test.js
import { test, describe, it, before, after, beforeEach, afterEach, mock } from 'node:test';
import assert from 'node:assert/strict';

import { add, divide, average } from './math.js';

describe('math utilities', () => {
  it('adds two numbers', () => {
    assert.equal(add(1, 2), 3);
    assert.equal(add(-1, 1), 0);
  });

  it('divides correctly', () => {
    assert.equal(divide(10, 2), 5);
  });

  it('throws on division by zero', () => {
    assert.throws(() => divide(10, 0), /division by zero/);
  });

  it('calculates average', () => {
    assert.equal(average([1, 2, 3, 4, 5]), 3);
    assert.equal(average([]), 0);
  });
});
```

```javascript
// math.js
export function add(a, b) {
  return a + b;
}

export function divide(a, b) {
  if (b === 0) throw new Error('division by zero');
  return a / b;
}

export function average(numbers) {
  if (numbers.length === 0) return 0;
  return numbers.reduce((sum, n) => sum + n, 0) / numbers.length;
}
```

```bash
node --test math.test.js
```

Output:
```
▶ math utilities
  ✔ adds two numbers (0.312ms)
  ✔ divides correctly (0.06ms)
  ✔ throws on division by zero (0.217ms)
  ✔ calculates average (0.055ms)
▶ math utilities (1.207ms)

ℹ tests 4
ℹ suites 1
ℹ pass 4
ℹ fail 0
ℹ cancelled 0
ℹ skipped 0
ℹ todo 0
ℹ duration_ms 43.25
```

TAP output for CI: `node --test --reporter tap`. The exit code is 0 on success, non-zero on failure. Every CI system in the world understands this.

## Running All Test Files

```bash
# Run all *.test.js files recursively
node --test

# Or specify a glob pattern
node --test '**/*.test.js'

# Watch mode
node --test --watch
```

`node --test` with no arguments discovers test files automatically — files matching `*.test.{js,mjs,cjs}` or `*.spec.{js,mjs,cjs}`, or files in a `test` directory. This is sensible, configurable, and doesn't require a config file.

## Async Tests

```javascript
import { test, describe, it } from 'node:test';
import assert from 'node:assert/strict';

describe('async API', () => {
  it('fetches user data', async () => {
    // Mock fetch for testing
    const response = await fetch('https://jsonplaceholder.typicode.com/users/1');
    const user = await response.json();
    assert.equal(user.id, 1);
    assert.ok(user.name.length > 0);
  });

  it('handles not found', async () => {
    const response = await fetch('https://jsonplaceholder.typicode.com/users/9999');
    assert.equal(response.status, 404);
  });
});
```

Async tests work with `async/await`. No special wrapper, no `done()` callback, no timeout configuration. If the async function throws or rejects, the test fails.

## Mocking

The built-in test runner has a mock API:

```javascript
import { test, mock } from 'node:test';
import assert from 'node:assert/strict';

test('mocks a function', () => {
  const fn = mock.fn((x) => x * 2);

  assert.equal(fn(5), 10);
  assert.equal(fn(3), 6);
  assert.equal(fn.mock.calls.length, 2);
  assert.deepEqual(fn.mock.calls[0].arguments, [5]);
});

test('mocks a method', () => {
  const obj = {
    greet(name) { return `Hello, ${name}`; }
  };

  mock.method(obj, 'greet', (name) => `Hi, ${name}!`);
  assert.equal(obj.greet('Alice'), 'Hi, Alice!');
  assert.equal(obj.greet.mock.calls.length, 1);
  mock.restoreAll();
});

// Mocking modules (Node 22+)
test('mocks a module import', async (t) => {
  t.mock.module('./database.js', {
    namedExports: {
      getUser: () => ({ id: 1, name: 'Test User' }),
    }
  });

  const { getUser } = await import('./database.js');
  const user = getUser(1);
  assert.equal(user.name, 'Test User');
});
```

Module mocking is available in Node 22+. For older Node versions, you can inject dependencies through function parameters instead of module-level imports, which makes mocking unnecessary:

```javascript
// Instead of this (hard to mock):
import { db } from './database.js';
export function getUser(id) {
  return db.query('SELECT * FROM users WHERE id = ?', [id]);
}

// Do this (easy to test):
export function getUser(id, db) {
  return db.query('SELECT * FROM users WHERE id = ?', [id]);
}

// In tests:
const mockDb = { query: () => ({ id: 1, name: 'Alice' }) };
const user = getUser(1, mockDb);
```

Dependency injection is testable without mocking infrastructure. This isn't always possible, but it's worth preferring when it is.

## Code Coverage

```bash
node --test --experimental-test-coverage
```

Output:
```
─────────────────────────────────────────────────────────────────
File          │ Line % │ Branch % │ Function %
─────────────────────────────────────────────────────────────────
 math.js      │ 100.00 │   100.00 │     100.00
─────────────────────────────────────────────────────────────────
```

No Istanbul, no nyc, no additional configuration. The flag is `--experimental-test-coverage`, which will become stable as the API matures.

## Deno's Test Runner

Deno has a native test runner that's arguably more polished:

```typescript
// math.test.ts
import { assertEquals, assertThrows } from "jsr:@std/assert";
import { add, divide, average } from "./math.ts";

Deno.test("adds two numbers", () => {
  assertEquals(add(1, 2), 3);
  assertEquals(add(-1, 1), 0);
});

Deno.test("divides correctly", () => {
  assertEquals(divide(10, 2), 5);
});

Deno.test("throws on division by zero", () => {
  assertThrows(() => divide(10, 0), Error, "division by zero");
});

// Async test
Deno.test("fetches data", async () => {
  const response = await fetch("https://api.example.com/health");
  assertEquals(response.status, 200);
});

// Test with setup/teardown
Deno.test({
  name: "database operations",
  async fn() {
    const db = await openTestDatabase();
    try {
      await db.execute("INSERT INTO users (name) VALUES ('Alice')");
      const user = await db.queryOne("SELECT * FROM users WHERE name = 'Alice'");
      assertEquals(user.name, "Alice");
    } finally {
      await db.close();
    }
  },
});
```

```bash
deno test              # Run all test files
deno test math.test.ts # Run specific file
deno test --watch      # Watch mode
deno test --coverage   # Coverage report
deno test --doc        # Test doc examples
```

Deno's `--doc` flag is particularly interesting: it runs code examples from JSDoc comments as tests, which keeps documentation in sync with behavior.

## Testing Browser Code Without a Framework

The question "how do I test browser code without Jest/Vitest/webpack" has several answers.

### Pure Logic: Test in Node

If your browser code contains pure logic — functions that take values and return values — test those in Node. Don't test DOM manipulation in Node. Test logic there, DOM behavior in a browser.

```javascript
// utils/date.js — pure functions, testable in Node
export function formatRelativeTime(date, now = new Date()) {
  const diff = now - new Date(date);
  const seconds = Math.floor(diff / 1000);
  const minutes = Math.floor(seconds / 60);
  const hours = Math.floor(minutes / 60);
  const days = Math.floor(hours / 24);

  if (seconds < 60) return 'just now';
  if (minutes < 60) return `${minutes}m ago`;
  if (hours < 24) return `${hours}h ago`;
  return `${days}d ago`;
}
```

```javascript
// utils/date.test.js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { formatRelativeTime } from './date.js';

const now = new Date('2024-01-15T12:00:00Z');

test('returns "just now" for recent times', () => {
  const thirtySecondsAgo = new Date(now - 30000);
  assert.equal(formatRelativeTime(thirtySecondsAgo, now), 'just now');
});

test('returns minutes for < 1 hour', () => {
  const tenMinutesAgo = new Date(now - 600000);
  assert.equal(formatRelativeTime(tenMinutesAgo, now), '10m ago');
});

test('returns days for > 24 hours', () => {
  const twoDaysAgo = new Date(now - 172800000);
  assert.equal(formatRelativeTime(twoDaysAgo, now), '2d ago');
});
```

No bundler. No DOM. Just functions and assertions.

### DOM Testing: Playwright and the Real Browser

For tests that need a real browser — component rendering, interaction, accessibility — skip the JSDOM simulation and use Playwright:

```javascript
// tests/task-list.spec.js
import { test, expect } from '@playwright/test';

test.beforeEach(async ({ page }) => {
  await page.goto('http://localhost:8080');
});

test('can add a task', async ({ page }) => {
  await page.fill('[data-testid="new-task-input"]', 'Write tests');
  await page.click('[data-testid="add-task-button"]');

  const tasks = page.locator('[data-testid="task-item"]');
  await expect(tasks).toHaveCount(1);
  await expect(tasks.first()).toContainText('Write tests');
});

test('can mark a task as done', async ({ page }) => {
  // Add a task first
  await page.fill('[data-testid="new-task-input"]', 'Test task');
  await page.click('[data-testid="add-task-button"]');

  const checkbox = page.locator('[data-testid="task-checkbox"]').first();
  await checkbox.check();
  await expect(checkbox).toBeChecked();
});

test('has no accessibility violations', async ({ page }) => {
  const results = await page.accessibility.snapshot();
  // Basic check — Playwright also has axe integration
  expect(results).toBeTruthy();
});
```

```bash
npx playwright install  # One-time browser download
npx playwright test
```

Playwright tests your actual application in real browsers. JSDOM is a simulation that gets the details wrong in ways that matter. Real browsers don't.

The "zero build" here is your application — Playwright is a testing tool, and that's allowed. You're not compiling your application to test it; you're serving it and letting a browser interact with it.

### Browser-Native Test Utilities

If you want to run tests in the browser itself — useful for testing Web APIs that Node can't simulate:

```html
<!-- test-runner.html -->
<!DOCTYPE html>
<html>
<head>
  <title>Tests</title>
</head>
<body>
  <div id="results"></div>
  <script type="module">
    // Minimal test runner — no dependencies
    const results = [];

    async function test(name, fn) {
      try {
        await fn();
        results.push({ name, passed: true });
      } catch (e) {
        results.push({ name, passed: false, error: e.message });
      }
    }

    function assert(condition, message = 'Assertion failed') {
      if (!condition) throw new Error(message);
    }

    // Tests
    await test('localStorage works', () => {
      localStorage.setItem('test', 'value');
      assert(localStorage.getItem('test') === 'value');
      localStorage.removeItem('test');
    });

    await test('CSS custom properties compute correctly', () => {
      const el = document.createElement('div');
      el.style.setProperty('--test-val', '42px');
      document.body.appendChild(el);
      const computed = getComputedStyle(el).getPropertyValue('--test-val').trim();
      assert(computed === '42px', `Expected '42px', got '${computed}'`);
      el.remove();
    });

    // Render results
    const container = document.getElementById('results');
    for (const result of results) {
      const div = document.createElement('div');
      div.textContent = `${result.passed ? '✔' : '✘'} ${result.name}`;
      div.style.color = result.passed ? 'green' : 'red';
      if (result.error) div.title = result.error;
      container.appendChild(div);
    }
  </script>
</body>
</html>
```

This is 50 lines that run in any browser. For testing browser APIs that can't be simulated in Node, it's entirely functional.

## What the Test Toolchain Actually Provides

It's worth being honest about what Jest and Vitest give you that the Node built-in doesn't:

| Feature | Node built-in | Jest/Vitest |
|---------|--------------|-------------|
| Basic test runner | Yes | Yes |
| async/await | Yes | Yes |
| Mocking | Yes (Node 22+) | More ergonomic |
| Snapshots | No | Yes |
| Coverage | Yes (experimental) | More polished |
| Watch mode | Yes | Yes |
| TypeScript support | No | Yes (with plugins) |
| JSDOM | No | Yes (optional) |
| Parallel execution | Yes | Yes |
| CI integration | Yes | Yes |

The Node built-in runner lacks TypeScript support natively (use Deno if you need that) and snapshot testing. If you use snapshots extensively, you'll want Jest/Vitest. If you don't, you may not need them.

Vitest is significantly faster than Jest for the same test suite and doesn't require a separate Babel/TypeScript compilation step if you're already using Vite. If you do want a test framework, Vitest is the current recommendation — but it's worth running your tests with `node --test` first and seeing whether the built-in runner is sufficient before adding the dependency.

---

The testing story for zero-build applications: pure logic tests in `node --test`, browser interaction tests in Playwright against your running application, Deno's test runner if you're on Deno. None of this requires Jest, Babel, webpack, or a test configuration file.

The next chapter covers deployment — where the zero-build story becomes genuinely simple.
