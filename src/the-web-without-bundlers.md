# The Browser Already Knows How to Load Files

*ES Modules Are Real and They Work*

---

Here is a complete web application:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>My App</title>
  <link rel="stylesheet" href="./styles.css">
</head>
<body>
  <div id="app"></div>
  <script type="module" src="./app.js"></script>
</body>
</html>
```

```javascript
// app.js
import { createRouter } from './router.js';
import { fetchUser } from './api.js';
import { renderDashboard } from './components/dashboard.js';

const router = createRouter();
const user = await fetchUser('/api/me');
renderDashboard(document.getElementById('app'), { user, router });
```

No webpack. No Vite. No esbuild. No `package.json`. No `node_modules`. No config files of any kind. You serve this directory and it works — in Chrome, Firefox, Safari, and Edge, on any device released in the last five years.

This chapter explains how. Not just that it works, but what the browser actually does when it loads a module, why it's been designed this way, and what this means for application architecture.

## How the Browser Loads Modules

When the browser encounters `<script type="module" src="./app.js">`, it does something fundamentally different from what it does with `<script src="./app.js">`.

With a classic script, the browser fetches the file and executes it. The script runs in the global scope. Variables declared at the top level are global. If two scripts declare `const user = ...`, they conflict. Load order determines what's available.

With a module script, the browser:

1. Fetches `app.js`
2. Parses it, looking for `import` statements *before executing any code*
3. Fetches all imported modules in parallel
4. For each fetched module, repeats step 2–3 recursively
5. Builds the complete dependency graph
6. Executes modules in dependency order (dependencies before dependents)
7. Executes `app.js` last

This is the **module loading algorithm**, and it has two important properties worth understanding.

### Modules Are Singletons

If two different modules both import `./utils.js`, the browser fetches it once and gives both importers a reference to the same module instance. This is guaranteed by the spec and implemented in every browser.

```javascript
// a.js
import { counter } from './counter.js';
counter.increment();

// b.js
import { counter } from './counter.js';
console.log(counter.value); // 1, not 0 — same instance

// main.js
import './a.js';
import './b.js';
```

```javascript
// counter.js
export const counter = {
  value: 0,
  increment() { this.value++; }
};
```

This is actually how you want shared state to work. CommonJS did this too (modules are cached after first require). The browser's native ESM does the same thing.

### Imports Are Static and Resolved Before Execution

The parser reads your import declarations before running a single line of code. This is intentional: it enables efficient parallel loading, makes circular dependency analysis possible, and prevents certain classes of bugs. It also means you can't put an import inside an if statement:

```javascript
// This is a syntax error
if (condition) {
  import { thing } from './thing.js'; // SyntaxError
}

// This is fine — dynamic import returns a Promise
if (condition) {
  const { thing } = await import('./thing.js');
}
```

The static analysis restriction on `import` declarations is a feature, not a limitation. It's what lets bundlers (when you use them) do tree shaking, and it's what lets browsers preload your dependency graph efficiently.

## The Module Specifier Problem

The one genuine friction point in native ESM is the module specifier. In Node and bundled environments, you write:

```javascript
import React from 'react';
import { format } from 'date-fns';
```

These are "bare specifiers" — module names without a path. They don't mean anything to the browser. The browser only understands:

- Relative paths: `'./utils.js'`, `'../lib/format.js'`
- Absolute paths: `'/src/utils.js'`
- Full URLs: `'https://esm.sh/react'`

Bare specifiers throw a `TypeError` in the browser. This is the main reason people thought "native ESM doesn't work for real applications." Import maps solve this (Chapter 4), but even without them, you can use full URLs:

```javascript
import React from 'https://esm.sh/react';
import { format } from 'https://esm.sh/date-fns';
```

This works. It's not ideal for large dependency trees, but for small projects it's perfectly functional. And if you want bare specifiers, one JSON file in your HTML gives you that.

## A Real Application Structure

Let's build something that has structure — multiple modules, shared utilities, real data fetching — without any build step.

```
myapp/
├── index.html
├── styles.css
├── app.js
├── router.js
├── api.js
└── components/
    ├── header.js
    ├── dashboard.js
    └── user-card.js
```

**`index.html`**:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Team Dashboard</title>
  <link rel="stylesheet" href="./styles.css">
</head>
<body>
  <div id="app"></div>
  <script type="module" src="./app.js"></script>
</body>
</html>
```

**`api.js`**:
```javascript
// Thin wrapper around fetch — handles JSON, base URL, errors
const BASE_URL = '/api';

async function request(path, options = {}) {
  const response = await fetch(`${BASE_URL}${path}`, {
    headers: { 'Content-Type': 'application/json', ...options.headers },
    ...options,
  });

  if (!response.ok) {
    throw new Error(`API error: ${response.status} ${response.statusText}`);
  }

  return response.json();
}

export const api = {
  getTeam: () => request('/team'),
  getUser: (id) => request(`/users/${id}`),
  updateUser: (id, data) => request(`/users/${id}`, {
    method: 'PATCH',
    body: JSON.stringify(data),
  }),
};
```

**`components/user-card.js`**:
```javascript
// Returns a DOM element — no framework, no JSX, just the DOM
export function UserCard({ name, role, avatar, onSelect }) {
  const card = document.createElement('article');
  card.className = 'user-card';
  card.innerHTML = `
    <img src="${avatar}" alt="${name}" width="48" height="48">
    <div class="user-info">
      <h3>${name}</h3>
      <p>${role}</p>
    </div>
  `;
  card.addEventListener('click', () => onSelect({ name, role }));
  return card;
}
```

**`components/dashboard.js`**:
```javascript
import { UserCard } from './user-card.js';

export function Dashboard({ container, team }) {
  container.innerHTML = '<h1>Team Dashboard</h1>';

  const grid = document.createElement('div');
  grid.className = 'team-grid';

  for (const member of team) {
    const card = UserCard({
      ...member,
      onSelect: (user) => console.log('Selected:', user),
    });
    grid.appendChild(card);
  }

  container.appendChild(grid);
}
```

**`app.js`**:
```javascript
import { api } from './api.js';
import { Dashboard } from './components/dashboard.js';

// Top-level await works in modules — no wrapper function needed
const team = await api.getTeam();

Dashboard({
  container: document.getElementById('app'),
  team,
});
```

This is a real application. It has data fetching, multiple modules, component composition, and DOM manipulation. It runs in the browser as-is. No compilation, no bundling, no toolchain.

## Top-Level Await

Notice `await api.getTeam()` at the top level of `app.js`. This works in modules. It would be a syntax error in a classic script.

Top-level await treats the module as an async function, but from the importing module's perspective, it still resolves before any code that depends on it runs. If `app.js` is the entry point, the browser simply waits for its top-level awaits to resolve before executing the module. If module B imports module A which uses top-level await, module B waits for module A to complete.

This means no more immediate invocation wrappers:

```javascript
// Before: classic scripts needed this
(async function() {
  const data = await fetchData();
  render(data);
})();

// Now: modules just do this
const data = await fetchData();
render(data);
```

Browser support: Chrome 89, Firefox 89, Safari 15, Edge 89. If you're targeting modern browsers, it's available.

## Module Scope vs. Global Scope

Modules have their own scope. Variables declared at the top level of a module are not global. This is different from classic scripts where top-level `var` declarations become properties of `window`.

```javascript
// classic.js (script)
var DEBUG = true; // window.DEBUG === true

// module.js (type="module")
const DEBUG = true; // not on window, not accessible from other scripts
```

This is strictly better behavior, but it means you can't expose module values to the console by accident the way you could with scripts. If you need something globally accessible during development, you can always do `window.myThing = myThing` explicitly, but you won't do it accidentally.

It also means: if you're refactoring a classic-script application to use modules, watch for code that depends on other scripts' globals. That pattern breaks — and breaking it is correct, and you should fix it properly.

## CORS and the Local Development Problem

There's one thing you need to know before you get to `file://` URLs and wonder why nothing works.

Modules loaded over `file://` URLs have restricted behavior because of CORS. The browser treats each file as a different origin, so cross-file imports fail. This is not a bug; it's CORS doing its job.

**You need a local HTTP server to develop with native ES modules.**

This is less painful than it sounds:

```bash
# Python (installed on almost everything)
python3 -m http.server 8080

# Node (if you have it)
npx serve .

# Deno
deno run --allow-net --allow-read https://deno.land/std/http/file_server.ts

# Caddy (if installed)
caddy file-server --listen :8080
```

Start one of these in your project directory and open `http://localhost:8080`. That's your entire dev setup. No webpack dev server, no config, no plugins. A directory and a file server.

## Performance: Bundlers vs. Native Loading

The question that should occur to you: if the browser makes one HTTP request per module, doesn't that get slow?

For small applications: no. HTTP/2 (which any modern server and CDN supports) multiplexes requests over a single connection. Fetching 20 small modules is not meaningfully slower than fetching one large bundle.

For large applications: yes, eventually. If you have hundreds of modules, the waterfall of module graph resolution can add latency. This is where bundling at deploy time (not during development) makes sense — but it's a deployment optimization, not a development requirement.

The ecosystem inverted this. Bundling became the default development experience, and unbundled development was the exception for production. Native ESM inverts it back: develop with the browser's actual module system, optimize for production separately if you need to.

A few practical data points:

- Google's [Squoosh](https://squoosh.app/) uses no bundler in development
- [Preact](https://preactjs.com/) is 3KB and works with native ESM directly
- Most "small to medium" applications (< 50–100 modules) have imperceptible load time differences between bundled and unbundled

The performance argument for bundling is real but often applied where it doesn't matter. A corporate internal tool that serves 50 users doesn't have a performance problem that requires webpack.

## Module Preloading

If you do have a deep module graph and want to preload it, there's a native mechanism:

```html
<link rel="modulepreload" href="./components/dashboard.js">
<link rel="modulepreload" href="./components/user-card.js">
<link rel="modulepreload" href="./api.js">
```

`modulepreload` tells the browser to fetch and parse these modules before they're imported, eliminating the waterfall. You get bundle-like performance without a bundler. This is supported in Chrome, Edge, and (as of 2022) Safari and Firefox.

## What You Get That Bundlers Can't Give You

Native ESM isn't just "bundlers but slower." It has properties that bundled code doesn't:

**Source maps are your actual source.** When you inspect a network error or debug in DevTools, the source you see is the actual file you wrote — not a minified bundle with a source map approximation. The module URL in an error stack trace is real and navigable.

**Module caching is semantically correct.** The browser caches modules by URL. If you cache-bust properly (by version in the URL), old modules expire correctly. With bundles, you cache-bust the whole bundle when any file changes.

**Incremental loading.** With dynamic `import()`, code only loads when it's needed. Bundlers simulate this with code splitting; native ESM does it natively with no configuration.

**Live development without HMR.** Browser caches modules for the session. A hard refresh (`Cmd+Shift+R`) gives you a clean slate. For many types of changes, this is all you need.

---

The browser's module system is not a subset of what bundlers provide. For many classes of application, it's a superset — semantically correct, debuggable, and free of configuration. The next chapter goes deeper into what native ESM looks like when you take it seriously as a production architecture.

What you've seen here is that the baseline capability — multiple modules, dependency resolution, top-level await, component composition — is available without any tooling at all. The browser knew how to load files all along. You just had to let it.
