# The Zero Build Movement

*A technical guide to building web applications without build systems.*

---

The browser can load ES modules natively. CSS has variables, nesting, and cascade layers. Import maps let you alias bare specifiers without npm. Deno runs TypeScript without compilation. Node has a built-in test runner. You can deploy a full web application as a static directory to a CDN.

None of this is new. Most of it has been available for four to six years. The question worth asking is: why are we still configuring webpack?

This book is an honest, working answer to that question — with real code, real trade-offs, and zero romanticism about complexity.

---

## Table of Contents

| Chapter | Title |
|---------|-------|
| Preface | Why This Book Exists |
| 1 | [The Build System You Didn't Ask For](./src/introduction.md) — A brief history of unnecessary complexity |
| 2 | [The Browser Already Knows How to Load Files](./src/the-web-without-bundlers.md) — ES Modules are real and they work |
| 3 | [Native ESM in the Browser: No Bundler Required](./src/native-esm.md) — Module loading, dynamic imports, and performance |
| 4 | [Import Maps: Dependency Management Without Node Modules](./src/importmaps.md) — Bare specifiers, CDN dependencies, version pinning |
| 5 | [Deno and the Zero Config Philosophy](./src/deno-zero-build.md) — URL imports, built-in TypeScript, no config required |
| 6 | [Modern CSS Without Sass](./src/css-without-preprocessors.md) — Custom properties, nesting, layers, and container queries |
| 7 | [HTML-First Development](./src/html-first.md) — Web components, declarative shadow DOM, and what you get for free |
| 8 | [Server-Side Zero Build](./src/server-side-zero-build.md) — Go, Deno, Bun, and single-binary deployments |
| 9 | [Testing Without a Toolchain](./src/testing-without-toolchains.md) — Node's built-in test runner, Deno's test runner, and browser-native testing |
| 10 | [Deploying Zero Build Apps](./src/deployments.md) — Static hosting, CDNs, and the absence of CI complexity |
| 11 | [When You Actually Need a Build System](./src/when-you-actually-need-a-build.md) — Be honest with yourself |
| 12 | [Real Projects, Zero Build](./src/real-world-projects.md) — Case studies that actually shipped |
| 13 | [The Build System You Keep](./src/conclusion.md) — Is the one you can justify |

---

## Building Locally

You need [mdBook](https://rust-lang.github.io/mdBook/guide/installation.html) installed.

```bash
# Install mdBook (requires Rust)
cargo install mdbook

# Build the book
mdbook build

# Serve with live reload at http://localhost:3000
mdbook serve --open
```

No npm. No node_modules. No webpack config. Just a Rust binary and some markdown.

---

## About This Book

Written by [Claude Code](https://claude.ai/code) for [CloudStreet](https://cloudstreet.dev).

The irony of using a build system (mdBook) to publish a book about not using build systems is acknowledged. mdBook compiles markdown to HTML. That is, charitably, its only job, and it does it without requiring you to understand Rollup.

---

## Contributing

Found an error? Have a better example? Open an issue or a PR. The source is markdown. You can read it without installing anything.

## License

[MIT](./LICENSE)
