# Server-Side Zero Build

*Go, Deno, Bun, and Single-Binary Deployments*

---

The zero-build movement on the server is, in some ways, older than on the client. Go has always compiled to a single binary. Rust compiles to a single binary. Even Java, for all its faults, produces a JAR that runs anywhere with a JVM. The "build step required" problem was mainly a JavaScript problem, created by Node's ecosystem of compilation and transpilation.

But even server-side JavaScript has options now. This chapter covers the full range.

## Go: The Original Zero-Config Server

Go was designed with operational simplicity as an explicit goal. The result is a language where:

- There are no runtime dependencies
- The output is a single static binary
- Cross-compilation is a first-class feature built into the toolchain
- The standard library handles HTTP, JSON, templates, crypto, and most common tasks

```go
// server.go — a complete JSON API in the standard library
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "strconv"
    "sync"
)

type Task struct {
    ID    int    `json:"id"`
    Title string `json:"title"`
    Done  bool   `json:"done"`
}

var (
    tasks  = []Task{}
    nextID = 1
    mu     sync.RWMutex
)

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("GET /api/tasks", func(w http.ResponseWriter, r *http.Request) {
        mu.RLock()
        defer mu.RUnlock()
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(tasks)
    })

    mux.HandleFunc("POST /api/tasks", func(w http.ResponseWriter, r *http.Request) {
        var body struct {
            Title string `json:"title"`
        }
        if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
            http.Error(w, "bad request", http.StatusBadRequest)
            return
        }

        mu.Lock()
        task := Task{ID: nextID, Title: body.Title}
        nextID++
        tasks = append(tasks, task)
        mu.Unlock()

        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(task)
    })

    mux.HandleFunc("PATCH /api/tasks/{id}", func(w http.ResponseWriter, r *http.Request) {
        id, err := strconv.Atoi(r.PathValue("id"))
        if err != nil {
            http.Error(w, "invalid id", http.StatusBadRequest)
            return
        }

        var updates struct {
            Done *bool   `json:"done"`
            Title *string `json:"title"`
        }
        json.NewDecoder(r.Body).Decode(&updates)

        mu.Lock()
        defer mu.Unlock()
        for i, t := range tasks {
            if t.ID == id {
                if updates.Done != nil {
                    tasks[i].Done = *updates.Done
                }
                if updates.Title != nil {
                    tasks[i].Title = *updates.Title
                }
                w.Header().Set("Content-Type", "application/json")
                json.NewEncoder(w).Encode(tasks[i])
                return
            }
        }
        http.Error(w, "not found", http.StatusNotFound)
    })

    log.Println("Listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

Note: the `"GET /api/tasks"` method-in-pattern syntax requires Go 1.22+, which shipped in February 2024.

```bash
# Run in development
go run server.go

# Build a binary
go build -o server server.go

# Cross-compile for Linux from Mac
GOOS=linux GOARCH=amd64 go build -o server-linux server.go

# The binary is self-contained — no runtime required on the target
scp server-linux user@myserver:~/
ssh user@myserver './server-linux &'
```

The binary is roughly 6MB for this program. It contains the Go runtime, your application code, and all dependencies. Deploy it anywhere Linux runs. No language installation, no package manager, no version conflicts.

The "build step" in Go is a single `go build` command that takes seconds. It doesn't require configuration, doesn't have a plugin ecosystem to manage, and produces the same output every time. This is the bar that the JavaScript build ecosystem should be measured against.

## Serving Static Files Alongside Your API

A common pattern: Go serves the API and the frontend from the same process:

```go
package main

import (
    "net/http"
    "log"
)

func main() {
    mux := http.NewServeMux()

    // API routes
    mux.HandleFunc("GET /api/", apiHandler)

    // Static files — everything else falls through to this
    static := http.FileServer(http.Dir("./public"))
    mux.Handle("/", http.StripPrefix("/", static))

    // For SPAs: 404 on assets should return 404, but 404 on routes
    // should return index.html. Handle this properly:
    mux.HandleFunc("GET /{path...}", func(w http.ResponseWriter, r *http.Request) {
        // Try to serve the file first
        // If it doesn't exist and has no extension, serve index.html
        http.ServeFile(w, r, "./public/index.html")
    })

    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

One binary. Serves the SPA and the API. Deploys to a $5/month VPS with `scp`. No reverse proxy required (though one is fine if you prefer). No Docker required (though one is fine if you prefer). No Kubernetes required (definitely not required).

## Bun: Fast JavaScript Without the Toolchain

Bun is a JavaScript runtime built with the goal of being fast — faster startup, faster execution, and a built-in HTTP server, test runner, bundler, and package manager.

For zero-build server development, the key properties are:

- Runs JavaScript and TypeScript natively (like Deno)
- No compilation step for TypeScript
- Fast startup time (relevant for serverless)
- `Bun.serve()` for HTTP — no framework needed for simple cases

```typescript
// server.ts — TypeScript, no compilation
interface Task {
  id: number;
  title: string;
  done: boolean;
}

const tasks: Task[] = [];
let nextId = 1;

const server = Bun.serve({
  port: 8080,

  async fetch(req) {
    const url = new URL(req.url);

    // CORS
    const headers = new Headers({
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',
    });

    if (req.method === 'OPTIONS') {
      return new Response(null, {
        headers: { 'Access-Control-Allow-Origin': '*',
                   'Access-Control-Allow-Methods': 'GET, POST, PATCH, DELETE',
                   'Access-Control-Allow-Headers': 'Content-Type' }
      });
    }

    if (url.pathname === '/api/tasks') {
      if (req.method === 'GET') {
        return Response.json(tasks, { headers });
      }

      if (req.method === 'POST') {
        const body = await req.json<{ title: string }>();
        const task: Task = { id: nextId++, title: body.title, done: false };
        tasks.push(task);
        return Response.json(task, { status: 201, headers });
      }
    }

    const match = url.pathname.match(/^\/api\/tasks\/(\d+)$/);
    if (match && req.method === 'PATCH') {
      const id = parseInt(match[1]);
      const index = tasks.findIndex(t => t.id === id);
      if (index === -1) return Response.json({ error: 'Not found' }, { status: 404, headers });

      const updates = await req.json<Partial<Task>>();
      tasks[index] = { ...tasks[index], ...updates };
      return Response.json(tasks[index], { headers });
    }

    return Response.json({ error: 'Not found' }, { status: 404, headers });
  },
});

console.log(`Listening on http://localhost:${server.port}`);
```

```bash
bun run server.ts  # Runs TypeScript directly
```

Bun's startup time is fast enough for serverless cold starts. Its HTTP throughput is competitive with Go for many workloads. The TypeScript support is native — no `tsc`, no `ts-node`, no compilation.

Unlike Deno, Bun has near-complete npm compatibility. If your application depends on npm packages, Bun is the path of least resistance for zero-build TypeScript server development.

## Deno Revisited: Production Server Patterns

Chapter 5 covered Deno's development story. Here's what production Deno deployment looks like.

### Single Binary with `deno compile`

```bash
deno compile \
  --allow-net \
  --allow-read=./public \
  --allow-env \
  --target x86_64-unknown-linux-gnu \
  --output app-linux \
  app.ts
```

The result is a ~70–90MB binary (it includes the Deno runtime) that runs your TypeScript application without any Deno installation on the target server. For operations teams who don't want to manage language runtimes, this is compelling.

### Serving Static Files with a Deno API

```typescript
// app.ts — API + static file server
import { Hono } from "jsr:@hono/hono";
import { serveStatic } from "jsr:@hono/hono/deno";

const app = new Hono();

// API routes
app.get('/api/status', (c) => c.json({ status: 'ok', time: new Date() }));

// Serve static files from ./public
app.use('/*', serveStatic({ root: './public' }));

// SPA fallback
app.get('/*', serveStatic({ path: './public/index.html' }));

Deno.serve({ port: 8080 }, app.fetch);
```

### Deno Deploy: Global Edge with Zero Config

For applications where you want global distribution without infrastructure:

```typescript
// This runs in 35+ regions worldwide with zero infrastructure config
import { Hono } from "jsr:@hono/hono";

const app = new Hono();

app.get("/api/geo", (c) => {
  // Deno Deploy provides request geolocation
  return c.json({
    country: c.req.header("x-forwarded-for"),
    region: Deno.env.get("DENO_REGION"),
  });
});

Deno.serve(app.fetch);
```

```bash
deployctl deploy app.ts
# → Live at https://your-app.deno.dev
```

No EC2, no ECS, no ALB, no Route53, no CloudFront. One command. Global.

## Node.js: Zero-Build Is Possible Here Too

Node has historically required compilation for TypeScript, but this is changing:

**Node 22.6+ with `--experimental-strip-types`:**

```bash
node --experimental-strip-types server.ts
```

Type annotations are stripped (not checked), the code runs. No tsc, no ts-node. This is experimental but shipping fast. The TypeScript checking happens in your editor via tsserver — the execution skips it.

**Node 23+ with `--experimental-transform-types`:**

```bash
node --experimental-transform-types server.ts
```

Adds support for TypeScript-only features (enums, namespaces, parameter properties) in addition to type stripping. More complete TypeScript support without compilation.

This matters because it means the zero-build philosophy can extend to existing Node applications — gradually, without a complete rewrite.

## SQLite Without an ORM Build Step

For data persistence in zero-build server applications, SQLite is underrated. It's a file, it's fast for reads, it handles reasonable write loads, and there are native bindings in every runtime:

```typescript
// Deno
import { Database } from "jsr:@db/sqlite";

const db = new Database("app.db");

db.exec(`
  CREATE TABLE IF NOT EXISTS tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    done INTEGER DEFAULT 0,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
  )
`);

const insertTask = db.prepare(
  "INSERT INTO tasks (title) VALUES (?) RETURNING *"
);

const getTasks = db.prepare("SELECT * FROM tasks ORDER BY created_at DESC");

export function createTask(title: string) {
  return insertTask.get(title);
}

export function listTasks() {
  return getTasks.all();
}
```

```go
// Go with the standard database/sql package and a SQLite driver
import (
    "database/sql"
    _ "modernc.org/sqlite"
)

db, _ := sql.Open("sqlite", "app.db")
db.Exec(`CREATE TABLE IF NOT EXISTS tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    done BOOLEAN DEFAULT FALSE
)`)
```

No ORM. No migration framework. No build step for the database layer. SQL is a query language — writing it directly is not a sin, and for small to medium applications it's the clearest approach.

## The Operational Case for Single Binaries

The argument for single-binary deployment isn't just laziness (though it's also laziness, and laziness is sometimes engineering wisdom):

**Reproducible deployments.** The binary on your server is exactly the binary you tested. No "works on my machine" with runtime version mismatches, no `npm install` on the server that could pull different packages.

**Simple rollback.** Keep the previous binary. If something breaks, swap them. Your rollback is a file copy.

**Minimal attack surface.** A Go binary running as a non-root user, with no package manager, no package cache, no npm on the server — this is a meaningfully smaller attack surface than a Node application with a `node_modules` directory containing hundreds of transitive dependencies, each potentially vulnerable.

**Cold start performance.** For containers and serverless, startup time matters. A Go binary starts in milliseconds. A Node application starting with hundreds of required modules starts measurably slower.

**Deployment simplicity.** `scp binary user@server:~/app && ssh user@server 'pkill app; ./app &'` is a complete deployment script. It's also embarrassingly fast.

---

The server-side zero-build stack in 2024:

- **Go**: Maximum performance, minimum runtime overhead, excellent standard library, true single binary
- **Deno**: TypeScript native, excellent developer experience, deploys to the edge as source or compiled binary
- **Bun**: Fastest Node-compatible runtime, TypeScript native, good for teams with npm dependencies
- **Node 22+**: Gradual path to zero-build TypeScript, largest ecosystem

All four of these support serving static files, handling JSON APIs, and connecting to databases without a build step. The "compile the server" step exists in Go — it's a single command, takes seconds, and produces a deployment artifact that's fundamentally simpler than anything npm-based.

The next chapter covers something you probably assumed required a complex toolchain: testing.
