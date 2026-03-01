# The Build System You Keep Is the One You Can Justify

---

A developer on a message board once described debugging a production issue that turned out to be caused by their build tool generating slightly different output depending on which CI runner picked up the job. The fix was to add deterministic build flags. The fix for the fix was to update their build tool version. The update broke three other things. They shipped the production fix two days after finding the root cause.

The root cause, by the way, was a null check.

This is the hidden cost of build systems: not just the configuration, not just the learning curve, not just the upgrade cycles — but the surface area for complexity-induced failures that live in the gap between your source code and what actually runs.

The zero-build approach eliminates that gap. What you write is what runs. The browser executes your JavaScript files. The network delivers your CSS. There is no intermediate representation, no generated artifact, no translation layer that could introduce subtle discrepancies between development and production.

---

## What This Book Covered

The case laid out across these chapters:

**The browser's module system is real and complete.** Native ES modules with static imports, dynamic imports, top-level await, `import.meta.url`, and live bindings — this is a full module system that has worked in every modern browser since 2017. You have been compiling your modules for a reason that stopped being true six years ago.

**Import maps solve the bare specifier problem.** One JSON blob in your HTML gives you the same bare-specifier ergonomics as npm, without npm. CDNs like esm.sh and jspm.io serve every major npm package as a proper ES module. You can have `import { format } from 'date-fns'` in the browser without a build step.

**Deno runs TypeScript natively.** Not "compiles TypeScript first" — runs it, directly, as source, with the Deno runtime handling type stripping transparently. The development environment and production environment are identical because they use the same runtime and the same source files.

**Modern CSS makes preprocessors optional for most applications.** Custom properties are runtime values that inherit and cascade, not compile-time constants. CSS nesting is in the spec and in every modern browser. `@layer` gives you explicit cascade control. Container queries make responsive components work correctly. The features that justified Sass now exist natively.

**HTML is an underused platform.** `<dialog>` has focus trapping and an accessible backdrop. The Popover API handles tooltips and dropdowns. Native form validation integrates with screen readers. Web components with Shadow DOM provide real encapsulation. Custom elements work in every modern browser. The amount of JavaScript that gets written to replace things HTML provides is significant and avoidable.

**Go, Deno, and Bun compile to single binaries.** Deployments that are one file, containing the runtime and your application code, deployable to any server without a language runtime installed. The operational simplicity of this is real.

**Node's built-in test runner works.** `node --test`, with no configuration, runs your test files, reports results, handles async, provides mocking (Node 22+), and produces TAP output for CI. Most projects don't need Jest.

**Zero-build deployment is simpler than build-heavy deployment.** A directory of files. Netlify, GitHub Pages, Cloudflare Pages, S3, a VPS with Caddy. No build step in CI means no build step that can fail in CI.

---

## The Mindset Shift

The practical knowledge in this book matters less than the underlying habit it's trying to install: **ask whether the platform already does the thing before reaching for a tool that does the thing.**

This sounds obvious. It isn't practiced consistently. The ecosystem's defaults — Create React App, Next.js, the proliferation of scaffolding tools — start with the assumption that you need a toolchain and work backward from there. The defaults are sticky. The defaults become the baseline. The baseline stops being questioned.

Questioning the baseline: that's the movement.

Not every application benefits from the zero-build approach. Chapter 11 covered the honest exceptions. The goal isn't to never use a build system. The goal is to use a build system because you need it, with a clear understanding of what problem it solves, rather than because it was in the starter template you copied five years ago.

---

## A Test For Your Current Stack

Open a terminal in your current project. Try to answer these questions:

1. Which build steps in your pipeline could you remove without affecting the production application?
2. If you wanted to add a new developer to the team, how long would it take them to get from `git clone` to a running development environment?
3. When did you last review your `package.json` dependencies? Which ones do you actually use?
4. Is your webpack config (or Vite config, or whatever config) maintained by the person who wrote it, or is it now an artifact that nobody wants to touch?
5. If your build tool released a major version tomorrow, what would it cost to upgrade?

The answers to these questions are diagnostic. If they're all "fine," then your build system is well-managed and proportionate to your needs. If any of them produce a wince, you know where to look.

---

## Where the Platform Is Going

The direction of the web platform is clearly toward reducing the build-vs-no-build distinction:

**CSS Houdini** gives JavaScript access to the CSS rendering pipeline — custom properties that paint, layout, and animate natively without JavaScript running in the main thread.

**The CSS `@scope` rule** and CSS nesting together approach the encapsulation that required Shadow DOM or CSS Modules.

**The module declarations proposal** would allow declaring multiple modules in a single file — reducing the HTTP request count without bundling.

**Import maps with `integrity`** would allow SRI hashing for import map entries, closing the security gap for CDN-hosted dependencies.

**Node's native TypeScript support** is shipping progressively. By the time you read this, `--experimental-strip-types` may be stable.

**Declarative Shadow DOM** is making server-side rendering of web components practical, which means the entire web component model becomes server-rendering-friendly.

The gap between "what the platform can do" and "what the ecosystem assumes requires tools" is closing. It closed significantly between 2017 and 2024. The trend continues.

---

## The Last Thing

Build systems are not the problem. Configuration files are not the enemy. The problem is cargo-cult adoption — using tools because they're familiar, because they were in the template, because they're what you learned first, without asking whether the problem they solve is the problem you have.

The zero-build approach is, at its core, a practice of asking that question. Every time a new tool appears in your stack, ask: what does this solve? Is that problem real in my context? Is this the simplest solution to that problem?

Sometimes the answer is "yes, we need this tool." Sometimes the answer is "the browser already does this." The important thing is that you asked.

You've been carrying build tools for longer than some of them were necessary. You don't have to put them all down. But now you know which ones you're keeping by choice, and which ones you were keeping out of habit.

That distinction is the whole point.

---

*The browser can load ES modules. It has for years. What else have you been compiling that you didn't have to?*
