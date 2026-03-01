# Deno and the Zero Config Philosophy

*URL imports, built-in TypeScript, no config required*

---

Deno is what happens when you build a JavaScript runtime from scratch in 2018, with the benefit of knowing what you'd do differently in Node.

Ryan Dahl, who wrote Node, gave a talk in 2018 called "10 Things I Regret About Node.js." The list included: `node_modules`, `package.json`, `require()` without extensions, and the complexity that had accumulated around what was supposed to be a simple thing. Deno is, in part, his correction.

The zero-config philosophy isn't accidental. It's the point.

## What Deno Does Differently

Start with what you don't need:

- No `package.json`
- No `node_modules` folder
- No `npm install`
- No Babel config
- No TypeScript config (TypeScript runs natively)
- No webpack/esbuild/Vite
- No `tsconfig.json` to placate

A Deno program:

```typescript
// server.ts — TypeScript, no compilation step
import { serve } from "jsr:@std/http/file-server";

serve({ port: 8080 });
```

Run it:

```bash
deno run --allow-net --allow-read server.ts
```

That's it. TypeScript runs directly. The standard library imports from JSR (JavaScript Registry). No install step. First run fetches and caches dependencies; subsequent runs use the cache.

## URL Imports and the JSR Registry

Deno's original dependency model used direct URL imports:

```typescript
import { assertEquals } from "https://deno.land/std@0.224.0/assert/mod.ts";
```

This is radical honesty: you're importing from a URL. The browser does this. HTTP is a package manager. The version is in the URL. The checksum is in the lockfile (`deno.lock`). There's no indirection through a local package store.

In 2024, Deno introduced [JSR](https://jsr.io) (JavaScript Registry) as a first-class package registry:

```typescript
// JSR imports — similar ergonomics to npm, but designed for ESM
import { parseArgs } from "jsr:@std/cli/parse-args";
import { join } from "jsr:@std/path";
import { Hono } from "jsr:@hono/hono";
```

JSR packages are TypeScript-first, publish source code (not compiled output), and work in Deno, Node, Bun, and browser environments. It's closer to what npm should have been if it had been designed after ES modules existed.

Node compatibility: Deno also supports `npm:` specifiers for when you need something from npm:

```typescript
import express from "npm:express";
import { z } from "npm:zod";
```

This runs the npm package in Deno's npm compatibility layer — no `npm install`, no `node_modules`. Deno fetches it, converts it if needed, and caches it.

## A Real Web Server in TypeScript, No Config

```typescript
// api.ts

interface User {
  id: number;
  name: string;
  email: string;
}

const users: User[] = [
  { id: 1, name: "Alice", email: "alice@example.com" },
  { id: 2, name: "Bob", email: "bob@example.com" },
];

function json(data: unknown, status = 200): Response {
  return new Response(JSON.stringify(data), {
    status,
    headers: { "Content-Type": "application/json" },
  });
}

function router(req: Request): Response {
  const url = new URL(req.url);

  if (url.pathname === "/api/users" && req.method === "GET") {
    return json(users);
  }

  const match = url.pathname.match(/^\/api\/users\/(\d+)$/);
  if (match && req.method === "GET") {
    const user = users.find((u) => u.id === Number(match[1]));
    return user ? json(user) : json({ error: "Not found" }, 404);
  }

  return json({ error: "Not found" }, 404);
}

Deno.serve({ port: 8000 }, router);
console.log("Listening on http://localhost:8000");
```

```bash
deno run --allow-net api.ts
```

This is a type-checked TypeScript HTTP server. No tsconfig. No compilation. No dependencies to install. The TypeScript runs directly in the Deno runtime. Errors are type errors, shown in the terminal, referencing your actual source file.

Compare the setup cost to an equivalent Node/TypeScript project:

```bash
# Node + TypeScript setup
npm init -y
npm install typescript ts-node @types/node express @types/express
npx tsc --init
# Edit tsconfig.json
# Write the server
# npm start or ts-node src/api.ts
```

vs:

```bash
# Deno setup
# Write the server
deno run --allow-net api.ts
```

The Deno version is smaller by literally every metric: lines of setup, files created, disk space used, things that can go wrong.

## Using Hono for Real HTTP Applications

For production HTTP handling, [Hono](https://hono.dev) is the recommended framework in the Deno ecosystem. It's fast, fully typed, and works across Deno, Bun, Node, and edge runtimes:

```typescript
// app.ts
import { Hono } from "jsr:@hono/hono";
import { cors } from "jsr:@hono/hono/cors";
import { logger } from "jsr:@hono/hono/logger";

interface Task {
  id: string;
  title: string;
  done: boolean;
  createdAt: string;
}

const app = new Hono();
const tasks = new Map<string, Task>();

app.use("*", logger());
app.use("/api/*", cors());

app.get("/api/tasks", (c) => {
  return c.json(Array.from(tasks.values()));
});

app.post("/api/tasks", async (c) => {
  const body = await c.req.json<{ title: string }>();
  const task: Task = {
    id: crypto.randomUUID(),
    title: body.title,
    done: false,
    createdAt: new Date().toISOString(),
  };
  tasks.set(task.id, task);
  return c.json(task, 201);
});

app.patch("/api/tasks/:id", async (c) => {
  const id = c.req.param("id");
  const task = tasks.get(id);
  if (!task) return c.json({ error: "Not found" }, 404);

  const updates = await c.req.json<Partial<Task>>();
  tasks.set(id, { ...task, ...updates });
  return c.json(tasks.get(id));
});

app.delete("/api/tasks/:id", (c) => {
  const id = c.req.param("id");
  if (!tasks.has(id)) return c.json({ error: "Not found" }, 404);
  tasks.delete(id);
  return c.body(null, 204);
});

Deno.serve({ port: 8000 }, app.fetch);
```

```bash
deno run --allow-net app.ts
```

Full CRUD REST API, typed, with CORS and logging middleware. One command to run. No config files. No install step on first run beyond Deno itself.

## The Permission Model

Deno's security model is explicit permissions. Programs can't read files, access the network, or spawn processes without being granted those permissions. This is uncomfortable for people used to Node's implicit "I can do anything" model, and it's exactly the right default.

Common permissions:

```bash
--allow-net              # All network access
--allow-net=api.github.com  # Only this host
--allow-read             # All file reads
--allow-read=/var/data   # Only this path
--allow-write=/tmp       # Only this path for writes
--allow-env              # Environment variables
--allow-run              # Spawning subprocesses
```

For development, `--allow-all` (`-A`) is the escape hatch. Don't use it in production without thinking.

The "aha" moment with permissions: when you run a third-party Deno script and it tries to make a network request your source code doesn't make, Deno stops and tells you. Compare this to Node, where a compromised npm package can exfiltrate your environment variables in silence. Deno's permission model isn't just inconvenience — it's a meaningful security boundary.

## deno.json: Minimal Configuration

Deno does have a config file, `deno.json`, but it's optional and its defaults are sensible:

```json
{
  "tasks": {
    "dev": "deno run --watch --allow-net --allow-read app.ts",
    "test": "deno test",
    "fmt": "deno fmt"
  },
  "imports": {
    "@hono/hono": "jsr:@hono/hono@^4.4.0",
    "@std/http": "jsr:@std/http@^0.224.0"
  }
}
```

The `imports` field in `deno.json` is effectively an import map for your project — you pin versions here and import by bare specifier everywhere else. Deno generates `deno.lock` to pin exact versions.

This is meaningfully simpler than a Node project's configuration surface area (`package.json`, `tsconfig.json`, `.eslintrc`, `.prettierrc`, `jest.config.js`, `babel.config.js`, `webpack.config.js`, `.env`, `.env.local`...). A Deno project at maximum needs two files: `deno.json` and `deno.lock`.

## File Watching Without Nodemon

Deno has `--watch` built in:

```bash
deno run --watch --allow-net app.ts
```

When any source file changes, Deno restarts automatically. No nodemon, no ts-node-dev, no configuration. The `--watch-exclude` flag excludes paths if needed.

## Testing Without Jest

Deno's built-in test runner:

```typescript
// api_test.ts
import { assertEquals, assertRejects } from "jsr:@std/assert";
import { app } from "./app.ts";

Deno.test("GET /api/tasks returns empty array initially", async () => {
  const req = new Request("http://localhost/api/tasks");
  const res = await app.fetch(req);
  assertEquals(res.status, 200);
  assertEquals(await res.json(), []);
});

Deno.test("POST /api/tasks creates a task", async () => {
  const req = new Request("http://localhost/api/tasks", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ title: "Write the book" }),
  });
  const res = await app.fetch(req);
  assertEquals(res.status, 201);
  const task = await res.json();
  assertEquals(task.title, "Write the book");
  assertEquals(task.done, false);
});
```

```bash
deno test
```

No configuration. Type-checked. Coverage available with `--coverage`. The test runner is built into the runtime the same way `deno fmt`, `deno lint`, and `deno doc` are. You don't need a separate toolchain for anything a modern project needs.

## Formatting and Linting Without Config

```bash
deno fmt          # Formats all TypeScript/JavaScript files
deno fmt --check  # CI: exits non-zero if files need formatting
deno lint         # Lints with sensible defaults
deno doc app.ts   # Generates documentation from JSDoc comments
```

`deno fmt` uses the same opinionated formatter regardless of project — no `.prettierrc` debates, no "tabs vs spaces" config variable, no project-specific formatting rules that new contributors need to discover. The formatter is the formatter.

This is the zero-config insight: the friction of agreeing on configuration is often more expensive than the value of the configuration. Opinionated defaults with no escape hatch are a feature.

## Compiling to a Single Binary

This is where Deno goes somewhere Node can't easily go:

```bash
deno compile --allow-net --allow-read app.ts -o app
./app  # Runs on any machine, no Deno required
```

`deno compile` produces a self-contained binary that includes the Deno runtime and your application code. You can ship this to a server with no runtime installed, no `npm install`, no dependency management. It's one file.

```bash
# Cross-compile for different targets
deno compile --target x86_64-unknown-linux-gnu app.ts -o app-linux
deno compile --target aarch64-apple-darwin app.ts -o app-mac-arm
deno compile --target x86_64-pc-windows-msvc app.ts -o app.exe
```

For the server chapter (Chapter 8), this is significant: a Go-style single-binary deployment from a TypeScript source. The operational simplicity is real.

## Deno Deploy: Serverless Without the Config

[Deno Deploy](https://deno.com/deploy) runs your Deno application at the edge in 35+ regions, globally, with:

- No infrastructure configuration
- No Docker
- No IAM roles
- No cold starts (it's actually fast)
- Free tier that's genuinely useful

```bash
# Install deployctl
deno install -gAf jsr:@deno/deployctl

# Deploy
deployctl deploy app.ts
```

Your application is live in under a minute. The "build step" for deployment is: there isn't one. Deno Deploy runs your TypeScript directly.

This is not just convenience. It's a different mental model: your development environment and your production environment are the same runtime, running the same source files. The discrepancy between "what runs on my machine" and "what runs in production" narrows to zero.

## The Honest Limitations

Deno is not Node. The npm ecosystem is vast, and while Deno's npm compatibility is good, not everything works:

- Native addons (`.node` files, bindings to C/C++ libraries) don't work
- Some packages that assume Node internals don't work or work poorly
- The Deno-native ecosystem is smaller than npm's

If your application depends heavily on packages that aren't available in pure JavaScript form, or if you have team members deeply invested in Node tooling, Deno is harder to adopt incrementally.

That said: most web applications, APIs, and tooling are pure JavaScript, and those work fine. The limitation is real but narrower than it appears.

---

Deno is the most complete realization of zero-config development: a runtime that runs TypeScript natively, manages dependencies via URLs and a lockfile, ships every tool you need as built-in commands, and compiles to single binaries for deployment. It didn't just skip the build step — it designed a runtime where the build step was never the right answer.

The next chapter returns to the browser, and to CSS — which has quietly become good enough to make preprocessors optional.
