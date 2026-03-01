# Deploying Zero Build Apps

*Static Hosting, CDNs, and the Absence of CI Complexity*

---

The deployment story for a zero-build application is short. Pleasantly, surprisingly short. No compilation step means no build step in CI. No build step in CI means no waiting for webpack to warm up. No webpack warmup means your CI pipeline goes from "four minutes on a good day" to "the time it takes to rsync files to a server."

Let's walk through the actual options.

## What You're Deploying

A zero-build frontend application is a directory. It looks like this:

```
public/
├── index.html
├── styles.css
├── app.js
├── router.js
├── api.js
├── components/
│   ├── header.js
│   ├── dashboard.js
│   └── user-card.js
└── assets/
    ├── logo.svg
    └── favicon.ico
```

This directory goes to a static host. Any static host. The technology doesn't matter. The files don't need to be processed. Netlify, Vercel, Cloudflare Pages, GitHub Pages, S3 + CloudFront, a VPS with Caddy, a Raspberry Pi with nginx — all of these work, and all of them work the same way.

## GitHub Pages

For projects already on GitHub, Pages is free and requires one YAML file:

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pages: write
      id-token: write

    steps:
      - uses: actions/checkout@v4

      # No build step. Upload the source directly.
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

      - id: deployment
        uses: actions/deploy-pages@v4
```

This workflow runs in under 30 seconds. There is no step labeled "Build" because there is no build. The time is spent on: checking out the repository and uploading the files. That's it.

If you're deploying from a repository root (no `public/` subdirectory), change `path: ./public` to `path: .` and add a `.nojekyll` file to prevent GitHub Pages from trying to interpret your site as a Jekyll site.

## Netlify

Netlify's default configuration assumes a build step. For zero-build apps, you override it:

**Option 1: `netlify.toml` in the repository root:**

```toml
[build]
  publish = "public"
  command = ""  # No build command

[build.environment]
  NODE_VERSION = "20"  # If you have any Node scripts

[[headers]]
  for = "/*"
  [headers.values]
    X-Content-Type-Options = "nosniff"
    X-Frame-Options = "DENY"
    Referrer-Policy = "strict-origin-when-cross-origin"

# SPA routing: all paths serve index.html
[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

**Option 2: Drag and drop.** Drop your `public/` directory on `app.netlify.com`. It deploys instantly. This is not a joke — it's the fastest way to share a working prototype.

Netlify gives you HTTPS, global CDN distribution, preview deployments for pull requests, and form handling — all free on the starter plan, all requiring zero infrastructure configuration from you.

## Vercel

Similar to Netlify. Create `vercel.json`:

```json
{
  "outputDirectory": "public",
  "buildCommand": "",
  "rewrites": [
    { "source": "/((?!api/).*)", "destination": "/index.html" }
  ],
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "no-store" }
      ]
    },
    {
      "source": "/(.*)\\.js",
      "headers": [
        { "key": "Content-Type", "value": "application/javascript" }
      ]
    }
  ]
}
```

The `rewrites` rule handles SPA routing: any request that doesn't start with `/api/` serves `index.html`, letting your client-side router take over.

## Cloudflare Pages

Cloudflare Pages is fast — genuinely, measurably fast. Content is served from Cloudflare's network, which means most users get their content from a datacenter 10–50ms away.

In the Cloudflare Pages dashboard:
- Build command: *(leave empty)*
- Build output directory: `public`

Or connect your GitHub repo and configure nothing else. Cloudflare detects that there's nothing to build.

For edge functions alongside static files:

```javascript
// functions/api/tasks.js — Cloudflare Pages Function
export async function onRequestGet({ env }) {
  const tasks = await env.TASKS_KV.list();
  return Response.json(tasks);
}

export async function onRequestPost({ request, env }) {
  const body = await request.json();
  const id = crypto.randomUUID();
  await env.TASKS_KV.put(id, JSON.stringify({ ...body, id }));
  return Response.json({ id, ...body }, { status: 201 });
}
```

Cloudflare Pages Functions run at the edge, globally, with sub-millisecond cold starts. The function code doesn't require a build step — Cloudflare deploys the JavaScript directly.

## S3 + CloudFront: The Enterprise Option

For organizations that need AWS, the zero-build deployment is simpler than it looks:

```yaml
# GitHub Actions workflow for S3 + CloudFront
name: Deploy to S3

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Sync to S3
        run: |
          aws s3 sync ./public s3://${{ secrets.S3_BUCKET }} \
            --delete \
            --cache-control "max-age=31536000,immutable" \
            --exclude "*.html" \
            --exclude "importmap.json"

          # HTML and import maps: no caching (they reference versioned assets)
          aws s3 sync ./public s3://${{ secrets.S3_BUCKET }} \
            --exclude "*" \
            --include "*.html" \
            --include "importmap.json" \
            --cache-control "no-cache"

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CF_DISTRIBUTION_ID }} \
            --paths "/*.html" "/importmap.json"
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: us-east-1
```

The caching strategy: JavaScript files and assets are cached aggressively (`max-age=31536000,immutable`) because they don't change — if you update a file, you rename it or add a content hash. HTML and import maps are never cached because they reference everything else.

For a zero-build app without a bundler providing content hashing, a simple approach is adding a query parameter version to your JS imports in `index.html`:

```html
<!-- index.html — update VERSION on each deploy -->
<script type="importmap">
{
  "imports": {
    "preact": "https://esm.sh/preact@10.22.1",
    "preact/hooks": "https://esm.sh/preact@10.22.1/hooks"
  }
}
</script>
<script type="module" src="./app.js?v=2024-11-15"></script>
```

Or automate it:

```bash
# In your deploy script, replace the version with the git commit hash
VERSION=$(git rev-parse --short HEAD)
sed -i "s/app\.js?v=[^\"']*/app.js?v=$VERSION/" public/index.html
```

## Self-Hosted with Caddy

For complete control, Caddy is a zero-config web server with automatic HTTPS:

```
# Caddyfile
example.com {
    root * /var/www/myapp
    file_server

    # SPA routing — try files, fall back to index.html
    try_files {path} /index.html

    # Cache headers
    @immutable {
        path *.js *.css *.svg *.ico *.woff2
    }
    header @immutable Cache-Control "max-age=31536000, immutable"
    header *.html Cache-Control "no-cache"

    # Security headers
    header {
        X-Content-Type-Options nosniff
        X-Frame-Options DENY
        Referrer-Policy strict-origin-when-cross-origin
    }
}
```

```bash
sudo caddy run  # Automatic HTTPS, HTTP/2, and HTTP/3
```

Caddy handles TLS certificate renewal automatically. HTTP/2 means modules load efficiently. The Caddyfile above is the entire server configuration — no nginx.conf with 200 lines of best-practices boilerplate.

Deployment to a VPS:

```bash
# On a fresh Ubuntu server
apt install -y caddy
mkdir -p /var/www/myapp
rsync -av --delete ./public/ user@server:/var/www/myapp/
```

Total configuration files: one Caddyfile. Total dependencies on the server: Caddy (a single binary).

## Content Security Policy for Native ESM

A zero-build application using CDN imports needs a Content Security Policy that allows those CDN origins:

```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  script-src 'self' https://esm.sh 'wasm-unsafe-eval';
  connect-src 'self' https://api.example.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
">
```

Or in HTTP headers (preferred):

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://esm.sh; connect-src 'self' https://api.example.com;
```

If you self-host all your dependencies, your CSP is simpler:

```
Content-Security-Policy: default-src 'self';
```

This is an argument for self-hosting your vendor files: better security posture, simpler CSP, no CDN dependency. The trade-off is that you maintain the files.

## MIME Types: The One Thing You Have to Check

Browsers require JavaScript modules to be served with `Content-Type: application/javascript`. Most web servers do this correctly by default. The one case where it goes wrong: object storage (S3, R2, GCS) where you uploaded the files without explicit MIME types.

If your modules fail to load with "incorrect MIME type" errors, check your server's headers:

```bash
curl -I https://example.com/app.js | grep content-type
# Should see: content-type: application/javascript
```

S3 typically gets this right if you upload via the console or aws CLI. If you're using a custom sync script, add explicit content type:

```bash
aws s3 cp app.js s3://bucket/app.js \
  --content-type "application/javascript"
```

## The CI/CD Pipeline for Zero-Build Apps

Here's a complete CI/CD pipeline for a zero-build application — frontend and a Deno backend:

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run frontend tests
        run: node --test 'src/**/*.test.js'

      - name: Run backend tests
        uses: denoland/setup-deno@v1
        with:
          deno-version: v1.x
      - run: deno test --allow-net api/

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Compile backend binary
        uses: denoland/setup-deno@v1
        with:
          deno-version: v1.x
      - run: |
          deno compile \
            --allow-net --allow-read --allow-env \
            --target x86_64-unknown-linux-gnu \
            --output api-server \
            api/main.ts

      - name: Deploy frontend to S3
        run: aws s3 sync ./public s3://${{ secrets.S3_BUCKET }} --delete
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Deploy API binary to server
        run: |
          scp api-server ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }}:~/
          ssh ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }} \
            'pkill api-server || true; ./api-server &'
```

This pipeline:
- Tests the frontend with Node's built-in test runner
- Tests the backend with Deno's test runner
- Compiles the backend to a single Linux binary
- Syncs the frontend to S3
- Deploys the compiled binary via SSH

Total build tools: `node --test` (built-in), `deno test` (built-in), `deno compile` (built-in), `aws s3 sync` (AWS CLI). No webpack, no Vite, no Babel, no Rollup, no PostCSS. The CI runs in about 90 seconds.

---

Zero-build deployment is not a compromise. It's often better than build-heavy deployment: faster CI, simpler configuration, fewer failure modes, and deployments that are trivially reversible (the previous artifact still exists; swap it back). The complexity of a build pipeline is technical debt. Every step is something that can break, something that needs maintenance, something that a new team member needs to understand.

The deploy pipeline you don't have is the one that never breaks at 4pm on a Friday.
