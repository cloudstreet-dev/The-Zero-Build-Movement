# The Build System You Didn't Ask For

*A Brief History of Unnecessary Complexity*

---

> "We had to bundle because the browser couldn't load modules."
> — Every tech lead, 2013–2022, about a problem that was solved in 2017.

---

Let's start with a question nobody was asked: at what point did "write some JavaScript and open it in a browser" become a multi-step process requiring a configuration file, a task runner, a transpiler, a module bundler, a dev server with hot module replacement, a source map generator, and a CI pipeline to run it all in production?

The answer is: gradually, then all at once, and then so completely that we forgot there was ever another way.

## 1995–2009: The Innocent Years

JavaScript started as a scripting language dropped directly into HTML. `<script src="app.js">`. The browser loaded it. It ran. If you needed two scripts, you wrote two script tags and hoped they loaded in the right order. This was genuinely annoying — global namespace pollution, load order dependencies, no encapsulation — but it was simple enough that a developer could hold the entire mental model in their head.

The tooling that emerged in this era was modest: CSS minifiers, JavaScript compressors, YUI Compressor (2007). You ran a script, it made your files smaller, you deployed the smaller files. One step. One tool. One problem solved.

## 2010–2012: The Module Problem Appears

CommonJS arrived with Node.js in 2009. Suddenly JavaScript had a real module system — `require()` and `module.exports`. This was great for server-side code. It was completely incompatible with the browser, which didn't have a module system at all.

Developers who wanted modules in the browser had two choices:

1. AMD (Asynchronous Module Definition) with RequireJS — wrap every module in a `define()` call and let RequireJS load them asynchronously. Syntactically ugly, but it worked.
2. Build-time bundling — take all your modules and concatenate them into a single file that worked in any browser.

Browserify (2011) made option 2 easy: write CommonJS modules as if you were writing for Node, run `browserify`, get a browser-compatible bundle. It was clever engineering solving a real problem. The problem was that the browser had no module system.

## 2013–2016: Complexity Compounds

Then several things happened at once, and they compounded each other into the ecosystem we have today:

**React shipped (2013)** and brought JSX with it. JSX is not JavaScript. It requires compilation. Babel emerged to handle this, and while it was at it, also transpiled the ES6 features that browsers hadn't implemented yet. Now you had a transpiler in your pipeline.

**Webpack arrived (2012, gained traction 2014)** and solved the bundling problem with more configurability than anyone needed. Where Browserify was a focused tool, webpack was a platform. You could transform anything into anything. CSS, images, fonts, markdown — all became "modules" that webpack could process. All required configuration.

**ES6 shipped in 2015** with classes, arrow functions, destructuring, template literals, and — critically — a native module syntax: `import` and `export`. The browser would eventually support this natively. It didn't matter: by the time browsers shipped ES modules in 2017, the ecosystem had fully committed to bundling. The infrastructure existed, the tutorials assumed it, the job postings required it. Bundling had become the default not because the browser couldn't handle modules, but because the ecosystem had organized itself around the assumption that it couldn't.

## 2017: The Moment That Should Have Changed Everything

In 2017, something important happened and nobody quite noticed.

Chrome 61 shipped native ES module support. Firefox 60 in 2018. Safari had already shipped it in 10.1 (2017). Edge 16, also 2017.

```html
<!-- This works. In every modern browser. Has for years. -->
<script type="module" src="./app.js"></script>
```

```javascript
// app.js — runs in the browser, no compilation, no bundling
import { formatDate } from './utils.js';
import { renderChart } from './chart.js';

const data = await fetch('/api/data').then(r => r.json());
renderChart(document.getElementById('chart'), data);
```

This works. It worked in 2017. It works now. No webpack. No Babel. No `node_modules`. No build step.

The ecosystem's response to this was, essentially, to continue doing what it was already doing.

## 2018–2023: The Tooling Metastasizes

The original problems — no native modules, missing ES6 syntax, browser fragmentation — were largely solved. New problems appeared to justify the existing infrastructure:

**Bundle size.** If you're sending a lot of JavaScript, tree shaking removes the code you don't use. Fair. But most applications became large enough to need tree shaking because they were consuming massive npm packages that were themselves designed for bundled environments. The tool created the problem it then solved.

**TypeScript.** Genuinely useful. Requires compilation. However: Deno runs TypeScript natively, and JSDoc types give you type checking in plain JavaScript. "We use TypeScript" stopped being sufficient justification for a build step.

**Framework requirements.** React's JSX isn't standard JavaScript. Vue's single-file components aren't standard HTML. These are framework-specific syntaxes that require tooling. But: Preact works with native ESM. Lit works with native ESM. Vue 3 works with native ESM. The framework requiring a build step is a property of the specific framework, not of "building web applications."

**Developer experience.** Hot module replacement, fast refresh, dev proxies — these are genuinely nice. They're also not production requirements. You can have a good development experience without webpack.

What happened between 2018 and 2023 is that the tooling layer got faster (Vite, esbuild, SWC), more opinionated (Create React App, Next.js, Nuxt), and more deeply embedded. The build step got harder to avoid, not because the underlying requirements changed, but because the ecosystem assumed it.

## What We're Actually Dealing With

Here's the honest accounting of what a modern JavaScript build pipeline does and whether you need it:

| Build step | Why it exists | Do you need it? |
|------------|---------------|-----------------|
| Module bundling | Browser had no native modules | No, since 2017 |
| Transpilation (ES6→ES5) | Browser compatibility | No, unless you support IE11 (you don't) |
| JSX compilation | JSX isn't valid JavaScript | Only if you use JSX |
| TypeScript compilation | TypeScript isn't valid JavaScript | Only if you use TypeScript |
| CSS preprocessing | CSS lacked variables, nesting | No, since 2022–2023 |
| Minification | Reduces file sizes | Yes, for production — one step |
| Tree shaking | Removes unused code | Only if you import large libraries |
| Code splitting | Loads code on demand | Available natively with dynamic `import()` |

Half this list hasn't been necessary for years. The other half is only necessary if you've made specific choices — usually choices that were themselves influenced by an ecosystem that assumed bundling.

## The Feedback Loop

The worst part isn't the complexity. It's the self-reinforcing nature of it.

New developers learn React with Create React App (or Vite). The tool hides all build configuration. They graduate to a job where webpack configuration exists. They need to understand it, modify it, maintain it. They learn webpack. They become the person who maintains the webpack config. When they write tutorials or start new projects, they reach for the tools they know. They don't question whether the tools are necessary because the tools have always been there.

This is not a conspiracy. Nobody decided to make web development complex. It emerged from a series of individually reasonable decisions made when the browser genuinely couldn't do what developers needed, and it persisted because ecosystems are sticky.

The people who wrote Browserify in 2011 were solving a real problem. The people who wrote webpack were solving real problems. The people who kept improving the tooling were making things genuinely better. And the result, now, is a default ecosystem configuration that is dramatically more complex than most applications require.

## The Counter-Argument You're Already Making

"But I need TypeScript." Maybe. JSDoc types give you type inference in VS Code without compilation, and `tsc --noEmit` checks your types without producing a build artifact. Deno runs TypeScript natively.

"But I need npm packages." Import maps let you use npm-compatible packages from CDNs like esm.sh and jspm.io in the browser, by bare specifier, without a local install.

"But tree shaking." If your application is large enough to meaningfully benefit from tree shaking, this book will tell you that honestly in Chapter 11. Most applications aren't.

"But deployment." A directory of HTML, CSS, and JavaScript files deploys to Netlify, Vercel, Cloudflare Pages, S3, or any web server in the world. Zero build configuration required.

## What This Book Is

This book is a systematic examination of what you can do without a build step, using the current web platform — not the web platform of 2016, but the one that exists right now in every modern browser.

It covers:

- Native ES modules and what the browser actually does when it loads them
- Import maps for dependency management without npm
- Modern CSS that makes preprocessors unnecessary for most use cases
- HTML capabilities that have shipped in the last five years
- Server-side development without a compilation step
- Testing without Jest, webpack, or a fifteen-minute CI bootstrap
- Deployment without a build pipeline
- Honest assessment of when you actually do need a build step

The goal isn't to convince you that build systems are always wrong. The goal is to make the decision conscious. Use a build system because you need it, not because it was the default.

---

In 2017, Chrome shipped ES module support. In that same year, most tutorial sites were still teaching `require()`. The gap between what the platform can do and what the ecosystem assumes it can do is the territory this book covers.

You've been carrying a build system for longer than you needed to. Let's see what's underneath it.
