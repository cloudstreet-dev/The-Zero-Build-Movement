# Preface

This book started, as many technical books do, with frustration.

Specifically: watching a colleague spend four hours configuring webpack to do something the browser has done natively since 2017. Four hours. For a feature that shipped in Chrome 61, Firefox 60, and Safari 10.1. The developer in question was smart, experienced, and completely unaware that what they were trying to do was already done.

That's not a failure of intelligence. That's a failure of the information environment around us. We've built an ecosystem where the default answer to every web development question is "add a build step," and we've been doing it so long that we've collectively forgotten to ask whether the browser can just... do the thing.

It frequently can.

---

This is not an anti-tooling screed. There are real reasons build systems exist, and Chapter 11 goes into them honestly. TypeScript is genuinely useful. Tree shaking matters at scale. If you're shipping a production React application to millions of users, you probably want a bundler. This book is not for that moment.

This book is for every other moment. The internal tool that doesn't need to support IE11 because IE11 is dead. The prototype that became production because it worked. The side project where the webpack config is longer than the application code. The team that spent more time on their CI pipeline than their users ever noticed.

For all of those moments: the browser is much more capable than the ecosystem's defaults suggest, and the gap between "what developers think requires a build step" and "what actually requires a build step" has been widening for eight years.

---

**What you'll find here:**

Real working code. Every example in this book runs without compilation — you can drop it in a directory, serve it, and it works. Where something is more complex, there's a working repository linked. Nothing here is theoretical.

Honest limits. The zero-build approach has real constraints. Each chapter names them. Where bundling is the right answer, the text says so.

Dry humor. Web development is inherently absurd in ways worth noting. The jokes are in service of the point, not instead of it.

---

**What you need to follow along:**

A modern browser (anything released in the last four years), a text editor, and a way to serve static files. `python -m http.server` works. So does `npx serve`. So does Caddy, nginx, or any CDN with an S3 bucket behind it. You do not need Node. You do not need npm. You do not need a package.json.

That last sentence will feel strange by the end of Chapter 2 and obvious by the end of Chapter 13. That's the goal.

Let's start at the beginning.
