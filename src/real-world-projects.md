# Real Projects, Zero Build

*Case Studies That Actually Shipped*

---

Theory is useful. Working code is more useful. This chapter covers the kinds of real applications that benefit from the zero-build approach, with enough specificity to understand the decisions made and why they held up.

The examples here are architectural patterns drawn from real categories of applications. The code runs. The trade-offs described are real.

## Case Study 1: Internal Analytics Dashboard

**The project**: A dashboard showing business metrics for a 20-person company. Live data, multiple chart types, filterable date ranges, exportable reports. Used by 15 people, all on modern browsers, on a corporate network.

**Why zero-build made sense**: No external users, no IE11 requirement, small team, no npm package dependencies beyond charting.

**Architecture**:

```
dashboard/
├── index.html
├── styles.css
├── app.js
├── api.js
├── router.js
├── components/
│   ├── chart.js       # Wraps a charting library
│   ├── table.js       # Data table with sorting/filtering
│   ├── date-picker.js # Date range selector using native <input type="date">
│   └── export.js      # CSV export
└── pages/
    ├── overview.js
    ├── revenue.js
    └── engagement.js
```

The charting library — [Recharts](https://recharts.org) was considered, but it requires React. Instead: [Chart.js](https://www.chartjs.org) ships as a proper ES module and works natively.

```html
<script type="importmap">
{
  "imports": {
    "chart.js": "https://esm.sh/chart.js@4.4.3",
    "chart.js/auto": "https://esm.sh/chart.js@4.4.3/auto"
  }
}
</script>
```

```javascript
// components/chart.js
import Chart from 'chart.js/auto';

export function createLineChart(canvas, { labels, datasets }) {
  return new Chart(canvas, {
    type: 'line',
    data: { labels, datasets },
    options: {
      responsive: true,
      plugins: {
        legend: { position: 'top' },
      },
      scales: {
        y: { beginAtZero: false },
      },
    },
  });
}

export function updateChart(chart, { labels, datasets }) {
  chart.data.labels = labels;
  chart.data.datasets = datasets;
  chart.update();
}
```

The data fetching layer talks to a Go API server that queries the database directly. The frontend imports the chart component when the relevant page loads:

```javascript
// pages/revenue.js
import { createLineChart, updateChart } from '../components/chart.js';
import { api } from '../api.js';

export async function RevenueView(container) {
  container.innerHTML = `
    <div class="page-header">
      <h1>Revenue</h1>
      <input type="date" id="start-date" class="date-input">
      <input type="date" id="end-date" class="date-input">
    </div>
    <canvas id="revenue-chart"></canvas>
  `;

  const canvas = container.querySelector('#revenue-chart');
  const startInput = container.querySelector('#start-date');
  const endInput = container.querySelector('#end-date');

  // Default: last 30 days
  const end = new Date();
  const start = new Date(Date.now() - 30 * 86400000);
  startInput.value = start.toISOString().slice(0, 10);
  endInput.value = end.toISOString().slice(0, 10);

  let chart;

  async function refresh() {
    const data = await api.getRevenue({
      start: startInput.value,
      end: endInput.value,
    });

    if (!chart) {
      chart = createLineChart(canvas, data);
    } else {
      updateChart(chart, data);
    }
  }

  startInput.addEventListener('change', refresh);
  endInput.addEventListener('change', refresh);
  await refresh();
}
```

**What worked**: Development was unusually fast. No build tooling to configure. Every change reloaded in the browser with a standard refresh. DevTools showed the actual source files, making debugging trivial.

**What was annoying**: Sharing mock data between the frontend and Go API required duplicating types — TypeScript would have helped with type-safe API responses. The team ultimately added JSDoc types and found them sufficient.

**Outcome**: Shipped in three weeks. Has been running for 18 months. The codebase is 2,800 lines of JavaScript. Zero npm dependencies. The CI pipeline runs in 25 seconds (tests + rsync to the server).

---

## Case Study 2: Documentation Site with Live Examples

**The project**: A documentation site for an open-source library. Static pages, searchable, with interactive code examples that users can edit and run.

**Why zero-build made sense**: Documentation sites are inherently content-heavy and read-only. The "interactive examples" requirement is the interesting constraint.

**Architecture**: Server-side rendered HTML for the main content (fast, SEO-friendly), with ES modules loaded on the client for the interactive features.

```html
<!-- The page HTML is pre-rendered markdown -->
<article class="docs-content">
  <h1>Getting Started</h1>
  <p>Install the library...</p>

  <!-- Interactive example: enhanced by JavaScript, readable without it -->
  <div class="example-container" data-code="example-1">
    <pre><code id="code-example-1">
import { createStore } from 'my-library';
const store = createStore({ count: 0 });
console.log(store.get('count')); // 0
    </code></pre>
    <div class="example-output" aria-live="polite"></div>
    <button class="run-button" type="button">Run</button>
  </div>
</article>
```

```javascript
// editor.js — loaded as a module, enhances the static content
export function initExamples() {
  const containers = document.querySelectorAll('.example-container');

  for (const container of containers) {
    const code = container.querySelector('code');
    const output = container.querySelector('.example-output');
    const button = container.querySelector('.run-button');

    // Make code editable
    code.contentEditable = 'true';
    code.spellcheck = false;

    button.addEventListener('click', async () => {
      const userCode = code.textContent;
      output.textContent = '';

      try {
        // Run the code in a sandboxed blob URL
        const blob = new Blob([
          `const console = {
            log: (...args) => self.postMessage({ type: 'log', args }),
            error: (...args) => self.postMessage({ type: 'error', args })
          };\n` + userCode
        ], { type: 'application/javascript' });

        const url = URL.createObjectURL(blob);
        const worker = new Worker(url, { type: 'module' });

        worker.onmessage = ({ data }) => {
          const line = document.createElement('div');
          line.className = data.type === 'error' ? 'output-error' : 'output-line';
          line.textContent = data.args.join(' ');
          output.appendChild(line);
        };

        worker.onerror = (e) => {
          const line = document.createElement('div');
          line.className = 'output-error';
          line.textContent = e.message;
          output.appendChild(line);
          worker.terminate();
        };

        // Clean up after 5 seconds
        setTimeout(() => {
          worker.terminate();
          URL.revokeObjectURL(url);
        }, 5000);

      } catch (e) {
        output.textContent = e.message;
      }
    });
  }
}
```

The code runner uses Web Workers and Blob URLs to execute user-submitted code in an isolated context. No `eval()` in the main thread. No server round-trip. The library being documented is itself available via import map.

**What worked**: The progressive enhancement approach meant the documentation was readable and useful before JavaScript loaded. The interactive examples added genuine value. Zero-build meant contributors could edit documentation locally with a static file server — no toolchain to install.

**What was annoying**: The code editor (a `contentEditable` div) lacks syntax highlighting. A proper editor like CodeMirror would require a bundled dependency. The team decided that basic highlighting via CSS was sufficient for their use case.

**Outcome**: The site deploys to GitHub Pages via a workflow that runs in 40 seconds. Contributors open PRs, the preview deploys automatically, and reviewers can see the result without installing anything.

---

## Case Study 3: Real-Time Collaboration Tool

**The project**: A shared task board for a remote team. Real-time updates via WebSocket, drag and drop, multiple users editing simultaneously.

**Why zero-build made sense**: The real-time requirement was served entirely by the browser's WebSocket API. The drag-and-drop requirement was served by the HTML Drag and Drop API. The remaining UI was modest enough not to need a framework.

**The WebSocket client**:

```javascript
// realtime.js
export function createRealtimeConnection(boardId) {
  const protocol = location.protocol === 'https:' ? 'wss:' : 'ws:';
  const ws = new WebSocket(`${protocol}//${location.host}/ws/boards/${boardId}`);

  const listeners = new Map();

  ws.onmessage = (event) => {
    const message = JSON.parse(event.data);
    const handlers = listeners.get(message.type) ?? [];
    for (const handler of handlers) {
      handler(message.payload);
    }
  };

  return {
    on(type, handler) {
      if (!listeners.has(type)) listeners.set(type, []);
      listeners.get(type).push(handler);
      return () => {
        const handlers = listeners.get(type);
        const index = handlers.indexOf(handler);
        if (index !== -1) handlers.splice(index, 1);
      };
    },

    send(type, payload) {
      ws.send(JSON.stringify({ type, payload }));
    },

    close() {
      ws.close();
    },
  };
}
```

**Native drag and drop**:

```javascript
// board.js
export function initDragAndDrop(board, onChange) {
  board.addEventListener('dragstart', (e) => {
    const card = e.target.closest('[data-card-id]');
    if (!card) return;
    e.dataTransfer.setData('text/plain', card.dataset.cardId);
    e.dataTransfer.effectAllowed = 'move';
    card.classList.add('dragging');
  });

  board.addEventListener('dragend', (e) => {
    e.target.closest('[data-card-id]')?.classList.remove('dragging');
  });

  board.addEventListener('dragover', (e) => {
    e.preventDefault();
    e.dataTransfer.dropEffect = 'move';
    const column = e.target.closest('[data-column-id]');
    column?.classList.add('drag-over');
  });

  board.addEventListener('dragleave', (e) => {
    const column = e.target.closest('[data-column-id]');
    if (column && !column.contains(e.relatedTarget)) {
      column.classList.remove('drag-over');
    }
  });

  board.addEventListener('drop', (e) => {
    e.preventDefault();
    const cardId = e.dataTransfer.getData('text/plain');
    const column = e.target.closest('[data-column-id]');
    if (!column) return;

    column.classList.remove('drag-over');
    onChange({ cardId, columnId: column.dataset.columnId });
  });
}
```

The native Drag and Drop API is verbose compared to a library like `react-beautiful-dnd`, but it works without any dependencies, handles touch with minor additions, and the verbosity is familiar once you've written it once.

**The server**: A Deno application running Hono with WebSocket support:

```typescript
// main.ts
import { Hono } from "jsr:@hono/hono";
import { upgradeWebSocket } from "jsr:@hono/hono/deno";

const app = new Hono();
const connections = new Map<string, Set<WebSocket>>();

app.get('/ws/boards/:boardId', upgradeWebSocket((c) => {
  const boardId = c.req.param('boardId');

  return {
    onOpen(_, ws) {
      if (!connections.has(boardId)) connections.set(boardId, new Set());
      connections.get(boardId)!.add(ws.raw!);
    },

    onMessage(event, ws) {
      // Broadcast to all other connections on this board
      const message = event.data.toString();
      const boardConnections = connections.get(boardId) ?? new Set();
      for (const conn of boardConnections) {
        if (conn !== ws.raw && conn.readyState === WebSocket.OPEN) {
          conn.send(message);
        }
      }
    },

    onClose(_, ws) {
      connections.get(boardId)?.delete(ws.raw!);
    },
  };
}));

// Serve static files
app.use('/*', serveStatic({ root: './public' }));

Deno.serve({ port: 8080 }, app.fetch);
```

**What worked**: Real-time collaboration in a few hundred lines of code. WebSocket is simple. The native Drag and Drop API worked well for desktop users.

**What was annoying**: Mobile drag and drop is painful with native APIs. The team added [Sortable.js](https://sortablejs.github.io/Sortable/) via import map for mobile, which is a reasonable trade-off.

**Outcome**: The server is a compiled Deno binary (~80MB) on a $6/month VPS. The frontend is static files on Cloudflare's CDN. Deployment is pushing a binary to the server and syncing the static directory. Total infrastructure: $6/month.

---

## Case Study 4: A Build Tool for a Zero-Build Shop

Here's a meta-example that the universe apparently demanded be included.

A small agency builds zero-build web applications. They have a standard setup that they clone for each new project: import map, CSS tokens, component patterns. They wanted to automate the project scaffolding.

The tool itself is a Deno script:

```typescript
// create-project.ts
import { parseArgs } from "jsr:@std/cli/parse-args";
import { exists } from "jsr:@std/fs";
import { join } from "jsr:@std/path";

const args = parseArgs(Deno.args, {
  string: ['name', 'template'],
  default: { template: 'basic' },
});

const projectName = args.name ?? args._[0]?.toString();
if (!projectName) {
  console.error('Usage: deno run -A create-project.ts --name my-project');
  Deno.exit(1);
}

const projectDir = join(Deno.cwd(), projectName);
if (await exists(projectDir)) {
  console.error(`Directory ${projectName} already exists`);
  Deno.exit(1);
}

await Deno.mkdir(projectDir, { recursive: true });
await Deno.mkdir(join(projectDir, 'src'));
await Deno.mkdir(join(projectDir, 'public'));

const indexHtml = `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>${projectName}</title>
  <link rel="stylesheet" href="/styles.css">
  <script type="importmap">
  {
    "imports": {
      "preact": "https://esm.sh/preact@10.22.1",
      "preact/hooks": "https://esm.sh/preact@10.22.1/hooks",
      "htm/preact": "https://esm.sh/htm@3.1.1/preact"
    }
  }
  </script>
</head>
<body>
  <div id="app"></div>
  <script type="module" src="/src/app.js"></script>
</body>
</html>`;

await Deno.writeTextFile(join(projectDir, 'public', 'index.html'), indexHtml);

const appJs = `import { html } from 'htm/preact';
import { render } from 'preact';
import { useState } from 'preact/hooks';

function App() {
  const [count, setCount] = useState(0);
  return html\`
    <main>
      <h1>${projectName}</h1>
      <p>Count: \${count}</p>
      <button onClick=\${() => setCount(c => c + 1)}>Increment</button>
    </main>
  \`;
}

render(html\`<\${App} />\`, document.getElementById('app'));
`;

await Deno.writeTextFile(join(projectDir, 'src', 'app.js'), appJs);

console.log(`Created ${projectName}`);
console.log(`cd ${projectName} && deno run --allow-net --allow-read jsr:@std/http/file-server public`);
```

```bash
# Install once
deno install -g -A create-project.ts

# Use
create-project my-new-app
cd my-new-app
deno run --allow-net --allow-read jsr:@std/http/file-server public
```

The tool that creates zero-build projects is itself a zero-build tool — a Deno script with no npm dependencies, no compilation, no build step.

---

## What These Projects Have in Common

Looking across these examples:

**The applications that worked well without bundling** shared these properties:
- Known, bounded user bases (internal tools, small consumer apps)
- Modern browser requirements (no IE11, often internal-only)
- Modest dependency requirements (2–10 external libraries, not 50)
- Clear separation between data-fetching logic and UI rendering
- Teams comfortable reading browser APIs documentation

**The trade-offs that consistently appeared**:
- TypeScript types via JSDoc is workable but more verbose than TypeScript syntax
- Mobile edge cases (touch events, drag and drop) sometimes needed libraries
- Native form styling is limited in some browsers
- The lack of a module bundler means `node_modules`-dependent packages require CDN adaptation

**What nobody missed**:
- Waiting for webpack to compile
- Debugging source maps that didn't match
- Webpack configuration files
- `npm install` adding 200MB of `node_modules` for a dependency tree nobody audited

---

Zero-build is not a constraint. It's a starting position that eliminates a category of complexity upfront. Most projects that start there stay there. The ones that outgrow it — because they hit the module graph size limit, because they need TypeScript features, because their dependency tree requires bundling — have a clear upgrade path.

Start without the build step. Add it when you can point to the specific problem it solves.
