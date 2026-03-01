# Import Maps: Dependency Management Without Node Modules

*Bare specifiers, CDN dependencies, version pinning — with one JSON blob*

---

The previous two chapters have been quietly avoiding a problem. Every import has used a relative path:

```javascript
import { formatDate } from './utils/date.js';
import { renderChart } from './components/chart.js';
```

What about third-party dependencies? In Node and bundled applications you write:

```javascript
import { format } from 'date-fns';
import confetti from 'canvas-confetti';
```

These are bare specifiers — names without paths. The browser doesn't know what `'date-fns'` means. It can't infer a URL from a package name. If you try this in a browser, you get:

```
Uncaught TypeError: Failed to resolve module specifier "date-fns".
Relative references must start with either "/", "./", or "../".
```

Import maps are the native solution to this. They're a JSON structure in your HTML that maps bare specifiers to URLs. One element, one JSON object, and you have the same bare-specifier ergonomics as npm — without npm.

## The Basic Syntax

```html
<!DOCTYPE html>
<html>
<head>
  <script type="importmap">
  {
    "imports": {
      "date-fns": "https://esm.sh/date-fns@3.6.0",
      "canvas-confetti": "https://esm.sh/canvas-confetti@1.9.3"
    }
  }
  </script>
</head>
<body>
  <script type="module" src="./app.js"></script>
</body>
</html>
```

```javascript
// app.js — this now works in the browser, no bundler
import { format, addDays } from 'date-fns';
import confetti from 'canvas-confetti';

const tomorrow = format(addDays(new Date(), 1), 'MMMM do');
document.getElementById('date').textContent = `Tomorrow: ${tomorrow}`;

document.getElementById('celebrate').addEventListener('click', () => {
  confetti({ particleCount: 100, spread: 70 });
});
```

The import map must appear before any module scripts. The browser processes it first and uses it to resolve specifiers when modules load. There can only be one import map per page.

## Path Mapping: Packages with Multiple Exports

Many packages have subpath exports — `lodash/get`, `react/jsx-runtime`, `date-fns/format`. Import maps handle this with trailing-slash mappings:

```json
{
  "imports": {
    "lodash/": "https://esm.sh/lodash-es/",
    "date-fns/": "https://esm.sh/date-fns/"
  }
}
```

```javascript
// Both of these work
import get from 'lodash/get';
import { format } from 'date-fns/format';
```

The trailing slash on both sides tells the browser: "for any specifier starting with `lodash/`, replace that prefix with `https://esm.sh/lodash-es/` and keep the rest." So `lodash/get` becomes `https://esm.sh/lodash-es/get`.

## Scoped Mappings

Different parts of your application might need different versions of the same package. Import maps support scoped resolution:

```json
{
  "imports": {
    "lodash": "https://esm.sh/lodash-es@4.17.21"
  },
  "scopes": {
    "/legacy/": {
      "lodash": "https://esm.sh/lodash@3.10.1"
    }
  }
}
```

Modules loaded from `/legacy/` get lodash 3. Everything else gets lodash 4. This is a power feature — you probably won't need it often, but when you're migrating a large application gradually, it's the right tool.

## CDNs for ESM Packages

To use packages in the browser without npm, you need them served as ES modules. Several CDNs do this:

**[esm.sh](https://esm.sh)** — Converts npm packages to ES modules on the fly. Most packages work. Supports subpath exports, TypeScript types, and version pinning.

```
https://esm.sh/react@18.3.1
https://esm.sh/preact@10.22.1
https://esm.sh/date-fns@3.6.0/format
```

**[jspm.io](https://jspm.io)** — Another npm-to-ESM CDN with a generator tool at [jspm.io/generator](https://jspm.io/generator) that builds the entire import map for you.

**[jsdelivr](https://www.jsdelivr.com)** (via the `/esm/` path) — Widely used CDN, ESM support for packages that publish ES modules.

**[unpkg](https://unpkg.com)** — Serves npm package files directly. Not all packages expose proper ES modules, but many do.

The practical recommendation: use esm.sh or jspm.io. Both are specifically designed for browser-native ES module consumption and handle the CommonJS-to-ESM conversion that most packages still need.

## Generating an Import Map

For a non-trivial dependency tree, writing the import map by hand is tedious and error-prone. The JSPM generator handles this:

```bash
# Install the JSPM CLI
npm install -g jspm  # Or: npx jspm

# Generate an import map for your dependencies
jspm install react preact date-fns lodash-es

# Output: importmap.json and updated index.html
```

Or use the web UI at [jspm.io/generator](https://jspm.io/generator) — paste in your package list, get an import map back. Copy it into your HTML.

The resulting import map includes not just your direct dependencies but their transitive dependencies, pinned to exact versions. This is what `package-lock.json` does, but as a JSON blob you include in HTML.

```json
{
  "imports": {
    "react": "https://esm.sh/react@18.3.1",
    "react-dom": "https://esm.sh/react-dom@18.3.1",
    "react-dom/client": "https://esm.sh/react-dom@18.3.1/client",
    "scheduler": "https://esm.sh/scheduler@0.23.2"
  }
}
```

## A Full Working Example: React Without npm

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>React Without npm</title>
  <script type="importmap">
  {
    "imports": {
      "react": "https://esm.sh/react@18.3.1",
      "react-dom/client": "https://esm.sh/react-dom@18.3.1/client",
      "htm": "https://esm.sh/htm@3.1.1"
    }
  }
  </script>
</head>
<body>
  <div id="root"></div>
  <script type="module">
    import { useState, useEffect } from 'react';
    import { createRoot } from 'react-dom/client';
    import htm from 'htm';

    const html = htm.bind(React.createElement);

    function Counter() {
      const [count, setCount] = useState(0);

      return html`
        <div>
          <p>Count: ${count}</p>
          <button onClick=${() => setCount(count + 1)}>+</button>
          <button onClick=${() => setCount(count - 1)}>-</button>
        </div>
      `;
    }

    const root = createRoot(document.getElementById('root'));
    root.render(html`<${Counter} />`);
  </script>
</body>
</html>
```

Wait — if React requires JSX, and JSX requires compilation, how is this working?

[HTM](https://github.com/developit/htm) (Hyperscript Markup Language) is a library from the Preact team that provides JSX-like syntax using tagged template literals. `html\`<${Component} />\`` is valid JavaScript that compiles to `React.createElement(Component)` at runtime. No Babel, no JSX transform, no build step.

This is a legitimate React application. It has hooks, state, event handlers. It runs in the browser as-is.

Note: `import React` isn't in the inline script — it's being accessed globally because `react` from esm.sh attaches to `globalThis.React`. A cleaner approach uses Preact, which has first-class ESM support:

```html
<script type="importmap">
{
  "imports": {
    "preact": "https://esm.sh/preact@10.22.1",
    "preact/hooks": "https://esm.sh/preact@10.22.1/hooks",
    "htm/preact": "https://esm.sh/htm@3.1.1/preact"
  }
}
</script>
```

```javascript
import { render } from 'preact';
import { useState } from 'preact/hooks';
import { html } from 'htm/preact';

function App() {
  const [count, setCount] = useState(0);
  return html`
    <div>
      <p>Count: ${count}</p>
      <button onClick=${() => setCount(c => c + 1)}>+</button>
    </div>
  `;
}

render(html`<${App} />`, document.getElementById('root'));
```

Preact with HTM is production-grade. The Preact team recommends it explicitly for build-free environments.

## Version Pinning and Integrity

One reasonable concern with CDN-hosted dependencies: what if the CDN changes the file? You're at the mercy of `esm.sh/date-fns@3.6.0` serving the same bytes tomorrow.

The answer is Subresource Integrity (SRI). You can add an `integrity` attribute to script tags, and the browser will refuse to execute the script if the hash doesn't match:

```html
<script type="module"
        src="https://esm.sh/preact@10.22.1"
        integrity="sha384-abc123...">
</script>
```

For import maps themselves, SRI support is in development (the `integrity` field in import maps is a proposed feature). In the meantime, your options are:

1. **Trust the CDN's version pinning.** `@10.22.1` should be immutable on any reputable CDN.
2. **Self-host your dependencies.** Download the ES module versions and serve them from your own CDN or static host.
3. **Use a lock file alternative.** Tools like Deno's `deno.lock` pin CDN dependencies by content hash.

Self-hosting is simpler than it sounds:

```bash
# Download the ESM version of a package
curl -o vendor/preact.js https://esm.sh/preact@10.22.1
curl -o vendor/preact-hooks.js https://esm.sh/preact@10.22.1/hooks
```

```json
{
  "imports": {
    "preact": "/vendor/preact.js",
    "preact/hooks": "/vendor/preact-hooks.js"
  }
}
```

Now you control the files. They're in your repo, they're version-controlled, and you can audit them. This is more work upfront, but it's the most conservative approach.

## Browser Support

Import maps are supported in:
- Chrome 89+
- Edge 89+
- Safari 16.4+
- Firefox 108+

As of 2024, all modern browsers support import maps. If you need to support older browsers, [es-module-shims](https://github.com/guybedford/es-module-shims) is a polyfill that implements import maps for browsers that don't have them natively:

```html
<script async src="https://esm.sh/es-module-shims@1.10.0"></script>
<script type="importmap">{ ... }</script>
```

es-module-shims also enables features like `modulepreload` polyfilling and JSON module assertions in browsers that don't support them yet.

## The `importmap.json` Pattern

For applications with many dependencies, keeping the import map inline in HTML gets unwieldy. A common pattern is to maintain `importmap.json` and inject it at build time — or, since this is zero-build, just reference it:

```javascript
// You can't reference an external import map in standard HTML yet,
// but you can generate the script tag dynamically at server render time,
// or use a simple server-side template.
```

Actually, the spec doesn't support `<script type="importmap" src="...">` — the import map must be inline. For server-rendered applications, this is fine: inject the JSON server-side. For static sites, you have two clean options:

1. Keep it inline in `index.html` and accept that `index.html` has a JSON blob in it.
2. Use a trivial build step — just a `cat importmap.json` injected into `index.html` — which doesn't require a bundler.

The purist answer: inline JSON in HTML is not a moral failing. It's fine.

## Comparing Import Maps to npm

| Feature | npm + bundler | Import maps |
|---------|--------------|-------------|
| Bare specifiers | Yes | Yes |
| Version pinning | package-lock.json | Import map JSON |
| Subpath exports | package.json exports | Trailing-slash mappings |
| Scoped resolutions | Not natively | `scopes` field |
| Transitive deps | Bundled automatically | CDN resolves, or manual |
| Type definitions | @types packages | .d.ts from CDN (esm.sh) |
| Offline dev | node_modules folder | Vendor files, or network |
| Auditing | `npm audit` | Manual / self-hosted |
| Tree shaking | Bundler does it | Not automatically |

The npm model wins on tooling support — the entire Node ecosystem assumes it. The import map model wins on simplicity: one JSON file, no local installs, no `node_modules` folder, no version conflicts between what's installed and what's in lockfile.

For applications that don't need tree shaking (most internal tools, small consumer apps, prototypes) and don't have complex dependency graphs, import maps are genuinely sufficient. For applications that depend on large libraries and care deeply about bundle size, you're going to want a build step eventually — and Chapter 11 will tell you when.

---

With import maps in hand, you have the three primitives of modern zero-build development: the browser's module system, dynamic imports for code splitting, and import maps for bare-specifier dependencies. The next chapter looks at Deno, which takes these primitives and builds an entire runtime philosophy around them.
