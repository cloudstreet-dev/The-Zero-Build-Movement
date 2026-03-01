# When You Actually Need a Build System

*Be Honest With Yourself*

---

Every book advocating for a simpler approach risks becoming a religion. The zero-build approach is not a religion. It's a tool appropriate for specific contexts, inappropriate for others, and the whole point is to make that distinction consciously rather than reflexively.

This chapter is where we draw the line honestly.

## When You Actually Need a Build System

### 1. Your TypeScript Uses TypeScript-Only Features

JSDoc types give you type inference in VS Code and type checking with `tsc --noEmit` without a build step. But JSDoc doesn't support every TypeScript construct:

- **Enums**: TypeScript `enum` doesn't exist in JavaScript. JSDoc can't express them.
- **Namespaces**: The `namespace` keyword is TypeScript-only.
- **Decorators** (the TypeScript/experimental form): Not yet in JavaScript.
- **`declare` blocks**: Ambient declarations for typing external things.
- **Parameter properties**: `constructor(private name: string)` — syntactic sugar that doesn't translate to JavaScript.

If you use these features, you need a TypeScript compilation step. The alternatives are:
- Use JSDoc where possible and accept the limitations
- Use Deno, which runs TypeScript natively
- Use Node 22's `--experimental-strip-types` for simple cases
- Use a build step for TypeScript and accept that trade-off

### 2. Your Application Has a Very Large Module Graph

This is the performance argument from Chapter 3, stated plainly.

If your application has 200+ modules in a graph several levels deep, the module loading waterfall will produce a measurably worse initial load time than a bundled build. The cascade of round trips adds latency that `modulepreload` mitigates but doesn't fully solve.

The threshold isn't precise, but a rough heuristic:
- **< 50 modules**: Native ESM is fine. Waterfall is not perceptible on any reasonable connection.
- **50–150 modules**: With `modulepreload` for the critical path, native ESM is acceptable.
- **150+ modules**: The waterfall is real. Bundling the initial load path makes sense.

The right question: who are your users and what are their connections? An internal tool used over a corporate LAN is different from a consumer app serving mobile users on 4G.

If you're in the "might need bundling" range, consider a hybrid approach: bundle only for production, develop unbundled. This is exactly what Vite does — Rollup-based bundling for production, native ESM dev server. You get both. The build step is confined to CI, not to your development loop.

### 3. You Need Tree Shaking for Bundle Size Reasons

Tree shaking requires static analysis of the module graph to determine which exports are actually used. Bundlers do this. Browsers don't.

If you import a large library and only use a fraction of it, you're shipping the whole thing without a bundler:

```javascript
// Without tree shaking, you get all of lodash-es (~130KB minified/gzipped)
import { debounce } from 'lodash-es';

// With tree shaking in a bundler, you get just debounce (~2KB)
```

The mitigating factors:
- CDNs like esm.sh do server-side tree shaking for some packages
- Modern browsers cache aggressively — if lodash-es is cached from another visit, the cost is near-zero
- 130KB gzipped is ~36KB — noticeable but not catastrophic for many applications

If you're serving millions of users and every kilobyte is measurable in conversion rates, you want tree shaking. If you're building an internal tool with 50 users on a fast network, this is not your bottleneck.

### 4. You're Using JSX

JSX is not JavaScript. `<Component prop="value" />` is a syntax error in a browser. To use JSX-based frameworks (React, Preact with JSX, Solid), you need a compilation step that transforms JSX to `createElement` calls.

The alternatives that don't require compilation:
- **HTM** (from the Preact team): `html\`<${Component} />\`` — tagged template literal JSX
- **Preact** with `h` function calls directly: `h(Component, { prop: 'value' })`
- **Lit** for web components: no JSX, template literals, no compilation required

For most applications that have adopted React for pragmatic reasons rather than JSX love, Preact + HTM is a viable replacement with an almost identical API and zero build step.

If your team is deeply invested in the React ecosystem (React DevTools, React-specific hooks and patterns, large numbers of React-specific npm packages), switching is a real cost. Maintain your React application, use Vite, and accept the build step. That's the honest answer.

### 5. You Need Dead Code Elimination for Security

If your codebase contains development-only code, admin-only code, or feature-flag-disabled code that shouldn't ship to production users for security reasons, you need a build step that removes it.

The hostname-check trick (`if (location.hostname === 'localhost')`) works for developer tooling but is unreliable for security-sensitive code. A determined user can change their host header or set an arbitrary hostname. Code that's shipped to the client is always inspectable.

If you have code that reveals business logic, internal admin capabilities, or security-sensitive behavior, that code needs to not exist in the production build — which means a build step to eliminate it.

### 6. You're Using CSS Preprocessors for Loops/Mixins

The CSS chapter covered what native CSS can do. Here's what it still can't do:

```scss
// Sass — no native CSS equivalent
@mixin truncate($lines) {
  overflow: hidden;
  display: -webkit-box;
  -webkit-line-clamp: $lines;
  -webkit-box-orient: vertical;
}

@for $i from 1 through 12 {
  .col-#{$i} {
    grid-column: span #{$i};
  }
}
```

Generating utility classes programmatically, parameterized mixins for shared patterns, complex conditionals in CSS — Sass still does these. Native CSS custom properties can approximate some of this, but not all.

If your CSS architecture depends on generated utility classes (you're building a design system, or using Tailwind), PostCSS or Sass is the right tool.

If your CSS architecture is component-scoped styles with design tokens, native CSS is likely sufficient.

### 7. You Need to Support Older Browsers

The zero-build approach targets modern browsers. "Modern" in 2024 means:
- Chrome/Edge 89+
- Firefox 108+
- Safari 15.4+

These browsers are a combined 95%+ of global web traffic. The remaining percentage includes:
- Internet Explorer: Extinct (Microsoft ended support June 2022)
- Chrome/Firefox/Safari versions 3+ years old: Rare but possible in enterprise, embedded, and certain regional markets

If your application has requirements to support browsers outside this range, you need transpilation. `tsc`, Babel, or SWC to compile down to older JavaScript feature sets. This is a real requirement for some applications and not something to dismiss.

The question to ask your stakeholders is not "can we drop IE11 support?" (IE11 is dead). It's "what's the actual browser usage distribution for our users?" Look at your analytics. If 2% of users are on iOS 14 and your application doesn't work there, that's a problem. If 0.1% are on a five-year-old Samsung browser, that's a judgment call.

### 8. You're Working on a Large Team with Strict Consistency Requirements

Build tools enforce consistency. A formatter that runs as part of the build fails the build for inconsistent code. A TypeScript compilation step fails for type errors. ESLint in the build pipeline blocks merges for lint failures.

You can achieve the same thing without a build pipeline — run formatters and linters in CI without bundling, use git hooks — but the organizational discipline required is real. Large teams with frequent code churn benefit from tools that make "broken code" hard to commit.

This isn't a technical reason to use a build system. It's an organizational one. That's still a valid reason.

## The Spectrum of Build Complexity

Not all build systems are equal. If you decide you need some build tooling, prefer the minimum:

**Level 0: No build** — Pure native, as covered in this book. Appropriate for most projects.

**Level 1: Single-purpose tool** — `tsc --noEmit` for type checking, `deno fmt` for formatting, or `csso` for CSS minification. One command, one purpose, no config.

**Level 2: Type stripping** — `node --experimental-strip-types` or Deno for TypeScript without full compilation. Still no bundling.

**Level 3: Light bundling** — esbuild for production bundling only. Sub-second build time, zero configuration, produces optimized bundles. Keep native ESM in development.

**Level 4: Vite** — Native ESM dev server, Rollup production builds. The sweet spot for applications that genuinely need bundling.

**Level 5: Full webpack** — Full configurability, plugin ecosystem, custom transforms. Worth the complexity only when you need something Vite can't provide.

Most projects that think they need Level 5 are actually at Level 1 or 2.

## The Self-Audit Questions

Before adding a build step, ask:

1. **What specific problem does this build step solve?** If the answer is vague ("better developer experience"), investigate whether that problem is real.

2. **How many users will notice the difference?** Tree shaking matters for millions of users. It doesn't matter for 50 internal users.

3. **What's the maintenance cost?** Every build tool is a dependency that can break. When webpack releases a breaking major version, someone on your team spends time on the upgrade.

4. **Could you achieve this differently?** JSX → HTM. Sass variables → CSS custom properties. Complex TypeScript → JSDoc + simpler types.

5. **Is this for development or production?** Development tooling (type checking, formatting) has no production cost. Production tooling (bundling, tree shaking) affects your deployment pipeline.

---

The honest conclusion: build systems are necessary for a meaningful fraction of web applications — specifically those with large module graphs, JSX requirements, significant TypeScript feature usage, or genuine performance constraints from bundle size.

They're unnecessary for a different, also meaningful fraction — internal tools, prototypes, small-to-medium consumer applications, documentation sites, and applications where the bottleneck isn't JavaScript bundle size.

The failure mode isn't using a build system. It's using a build system without asking whether you need one.
