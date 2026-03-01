# Native ESM in the Browser: No Bundler Required

*Module loading, dynamic imports, and performance — without a build step*

---

The previous chapter showed a working application. This one goes deeper — into how the browser's module system actually behaves, what you can do with it that bundlers simulate awkwardly, and where the interesting edges are.

## Static vs. Dynamic Imports

Native ESM gives you two ways to import modules, and they have meaningfully different semantics.

**Static imports** appear at the top of a file, are resolved before any code runs, and create a live binding to the exported values:

```javascript
import { formatDate, parseDate } from './date-utils.js';
```

**Dynamic imports** are expressions that return a Promise, can appear anywhere in your code, and are the native answer to code splitting:

```javascript
// Load a module only when a button is clicked
document.getElementById('load-chart').addEventListener('click', async () => {
  const { renderChart } = await import('./chart.js');
  renderChart(data);
});
```

Bundlers simulate dynamic imports with code splitting. With native ESM, it just works. The browser fetches `chart.js` (and its dependencies) when the import expression resolves, caches it, and you're done. No webpack magic numbers, no chunk file naming configuration, no async chunk loading infrastructure. It's a Promise that resolves to a module.

### Conditional Module Loading

Dynamic imports are how you do conditional loading without a build step:

```javascript
// Load the right locale module
const locale = navigator.language.split('-')[0];
const { messages } = await import(`./i18n/${locale}.js`).catch(() =>
  import('./i18n/en.js') // Fallback to English
);
```

```javascript
// Polyfill only when needed
if (!('IntersectionObserver' in window)) {
  await import('./polyfills/intersection-observer.js');
}
```

```javascript
// Development-only tooling
if (location.hostname === 'localhost') {
  const { setupDevTools } = await import('./dev/tools.js');
  setupDevTools();
}
```

That last one deserves attention. With a bundler, shipping dev tooling to production requires configuration — environment variables, build-time dead code elimination, tree shaking. With native ESM and a hostname check, the dev code simply never loads in production because the import expression never executes. No bundler, no config, no env vars.

## Live Bindings: ESM's Unexpected Feature

One of ESM's less-discussed properties is that exported bindings are *live*. When a module exports a variable and that variable changes, importers see the new value.

```javascript
// counter.js
export let count = 0;

export function increment() {
  count++; // This updates the exported binding
}
```

```javascript
// main.js
import { count, increment } from './counter.js';

console.log(count); // 0
increment();
console.log(count); // 1 — the binding updated
```

This is different from CommonJS, where `module.exports.count` would be a snapshot at import time. With ESM, you're importing a reference to the binding, not a copy of the value.

In practice, this matters most for:

- **Module-level state** shared across multiple importers
- **Circular dependencies** (live bindings make them tractable)
- **Re-exported values** from other modules

It's the kind of detail that doesn't matter until it does, and when it does, it explains behavior that would otherwise look like a bug.

## Import Assertions and Module Types

The browser's module system has extended beyond JavaScript. You can import JSON natively:

```javascript
import config from './config.json' with { type: 'json' };
console.log(config.apiUrl);
```

Support: Chrome 91+, Edge 91+, Firefox 127+, Safari 17.2+. The syntax changed from `assert` (old) to `with` (current) — use `with`.

CSS modules are in development and shipping progressively, but the intent is:

```javascript
import styles from './component.css' with { type: 'css' };
document.adoptedStyleSheets = [styles];
```

For now, CSS in modules is better handled with Constructable Stylesheets or just plain `<link>` tags. But the direction is clear: the browser's module system is becoming a general import mechanism for web resources, not just JavaScript files.

## Module Workers

Web Workers support ES modules, which means you can write modern, modular worker code without a bundler to transform it:

```javascript
// main.js
const worker = new Worker('./processor.worker.js', { type: 'module' });
worker.postMessage({ data: largeArray });
worker.onmessage = (e) => console.log('Result:', e.data);
```

```javascript
// processor.worker.js
import { expensive } from './computations.js';

self.onmessage = (e) => {
  const result = expensive(e.data.data);
  self.postMessage(result);
};
```

The Worker receives `type: 'module'` and gets the full ESM feature set — imports, top-level await, all of it. Without bundlers, this was historically impossible because bundlers were what made workers work with module syntax. Now it's a `{ type: 'module' }` option.

Support: Chrome 80+, Edge 80+, Safari 15+, Firefox 114+.

## Service Workers and ESM

Service workers are the exception — they don't support ES modules in all environments yet, and the story is complicated. Firefox added support in 2023. Chrome and Safari had it earlier. If you're writing a service worker, you may need to keep it as a classic script or use a minimal build step for that file alone.

This is an honest limit. Service workers have a separate module loading context, and it took longer to standardize and implement. If your architecture depends heavily on service workers, keep them in classic script format for now, and import the modules your service worker needs via dynamic import if needed.

## The Import Chain: What Actually Happens in DevTools

Open DevTools, go to the Network tab, and load a page with native ES modules. What you'll see is illuminating.

Each module appears as a separate request, with:
- Request type: `script`
- Initiator: the file that imported it
- Priority: High (for statically imported modules)

The requests aren't sequential. The browser discovers all direct imports in a file, fires them in parallel, and then processes their imports. The parallelism is good. The cascade — waiting for module A to load before discovering that module A imports modules B and C — is the cost.

For a module graph that's three levels deep, you have three round trips before all code is loaded, even if every file is tiny and your server is local. This is why `modulepreload` exists, and it's why very large unbundled applications can feel slow on the initial load.

A worked example: an application with 30 modules in a graph 4 levels deep, each module 2KB, on a 100ms RTT connection. Four round trips × 100ms = 400ms baseline latency, even though all 30 modules are 60KB total. Bundled into one file: 60KB, one round trip, 100ms. The bundle wins on initial load.

This is where the build-vs-no-build trade-off becomes real. More on this in Chapter 11.

## A Complete Module Architecture Example

Here's how a real, unbundled application might organize its modules. This is for a project management tool — not toy complexity:

```
src/
├── main.js                  # Entry point, top-level await
├── router.js                # Client-side routing
├── store.js                 # Shared application state
├── api/
│   ├── index.js             # Re-exports all API functions
│   ├── projects.js          # Project CRUD
│   ├── tasks.js             # Task CRUD
│   └── users.js             # User management
├── components/
│   ├── app.js               # Root component
│   ├── nav.js               # Navigation
│   ├── project-list.js      # Project listing view
│   ├── project-detail.js    # Project detail view (lazy)
│   ├── task-board.js        # Task board (lazy)
│   └── settings.js          # Settings page (lazy)
└── utils/
    ├── date.js              # Date formatting
    ├── dom.js               # DOM helpers
    └── events.js            # Event emitter
```

```javascript
// main.js
import { initRouter } from './router.js';
import { initStore } from './store.js';
import { App } from './components/app.js';

// These are statically imported and load in parallel
const store = await initStore();
const router = initRouter();

App({ mount: document.getElementById('app'), store, router });
```

```javascript
// router.js
export function initRouter() {
  const routes = {
    '/': () => import('./components/project-list.js'),
    '/projects/:id': () => import('./components/project-detail.js'),
    '/projects/:id/board': () => import('./components/task-board.js'),
    '/settings': () => import('./components/settings.js'),
  };

  // Simple pattern matching router
  async function navigate(path) {
    const [pattern, loader] = Object.entries(routes)
      .find(([p]) => matchPath(p, path)) ?? [];

    if (!loader) return renderNotFound();

    const { default: Component } = await loader(); // Dynamic import
    const params = extractParams(pattern, path);
    Component({ mount: document.getElementById('main'), params });
  }

  window.addEventListener('popstate', () => navigate(location.pathname));
  return { navigate };
}
```

The lazy-loaded components — `project-detail.js`, `task-board.js`, `settings.js` — only load when their route is visited. This is native code splitting. No webpack configuration, no Vite plugin, no chunk strategy. Just `import()`.

The statically imported modules — `router.js`, `store.js`, `components/app.js` — load in parallel at startup because the browser reads all static imports before executing any code.

## ESM and the `type: "module"` Script Tag Differences

Beyond just loading modules, `<script type="module">` has several behavioral differences from classic `<script>`:

**Deferred by default.** Module scripts never block HTML parsing. They're equivalent to `<script defer>` — the HTML is parsed in full before the script executes. You don't need `defer` or `DOMContentLoaded` wrappers.

**Executed once.** If you include the same module script twice, it only executes once. The second reference is a no-op.

**Strict mode.** Module scripts run in strict mode always. You can't use `with`, can't access `arguments.caller`, can't do a lot of the things you shouldn't be doing anyway.

**Cross-origin.** Modules fetched from other origins require CORS headers (`Access-Control-Allow-Origin`). Classic scripts don't. This is why CDN-hosted ESM packages need to send the right headers — and they do, if they're designed for it.

**`import.meta`** is available. This is a module-specific meta-object:

```javascript
// The URL of the current module
console.log(import.meta.url);
// "https://example.com/src/components/nav.js"

// Resolve a URL relative to this module
const dataUrl = new URL('../data/config.json', import.meta.url);
```

`import.meta.url` is surprisingly useful. It lets you reference files relative to the current module without knowing where the module is in the URL structure — the same problem that `__dirname` solves in CommonJS, solved natively.

```javascript
// Load a worker relative to this module's location
const workerUrl = new URL('./processor.worker.js', import.meta.url);
const worker = new Worker(workerUrl, { type: 'module' });
```

## Circular Dependencies

Circular dependencies are possible with ESM and handled correctly because of live bindings. If module A imports from B and B imports from A, the browser:

1. Starts loading A
2. Discovers A imports B, starts loading B
3. Discovers B imports A — but A is already being loaded
4. Gives B a reference to A's (currently incomplete) binding table
5. Finishes loading B, executing it (A's exports are undefined at this point)
6. Finishes loading A, filling in A's exports
7. B's binding references to A now see the correct values

This means circular dependencies "work," but with a caveat: if B's initialization code directly accesses A's exports at module evaluation time (not in a function called later), those exports are undefined. If B's code only accesses A's exports inside functions that are called after full initialization, everything works correctly.

The practical rule: circular dependencies are fine if both modules export functions that call each other. They're fragile if either module runs code at the top level that depends on the other.

```javascript
// This is fine — functions reference each other, calls happen after init
// a.js
import { b } from './b.js';
export function a() { return b() + 1; }

// b.js
import { a } from './a.js';
export function b() { return 42; } // doesn't call a() at module init time
```

```javascript
// This is fragile — B uses A's export at initialization time
// a.js
import { value } from './b.js';
export const a = value + 1; // value is undefined when b.js initializes

// b.js
import { a } from './a.js';
export const value = a * 2; // a is undefined here, value becomes NaN
```

The spec defines this precisely. Bundlers have to simulate it. Native ESM in the browser implements it correctly.

---

Native ESM is a real, complete module system. It's not a preview or a polyfill or a subset of what you get from webpack. In some ways — live bindings, `import.meta.url`, native dynamic import — it has capabilities that bundlers have to approximate.

The constraint is the HTTP loading model, and the next chapter addresses that directly: how import maps let you use bare specifiers without npm, and how to manage dependencies in a zero-build world.
