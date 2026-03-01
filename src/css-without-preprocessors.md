# Modern CSS Without Sass

*Custom properties, nesting, layers, container queries — these are just CSS now*

---

Sass was created in 2006 to solve real problems with CSS: no variables, no nesting, no ability to split stylesheets into logical files without HTTP overhead, limited calculation support. These were genuine limitations, and Sass (and its successors, Less, Stylus, PostCSS) addressed them.

Here's what's happened since then:

- CSS custom properties (variables): shipped 2015–2016, fully supported since 2017
- CSS nesting: shipped 2023, fully supported in all modern browsers
- CSS `@layer`: shipped 2022, fully supported
- CSS `calc()` and `clamp()`: shipped 2013–2019, fully supported
- CSS container queries: shipped 2023, fully supported
- CSS `color-mix()`, `oklch()`, relative colors: shipped 2023–2024

The features that justified Sass are now in CSS. The preprocessor is solving a problem that was closed.

## CSS Custom Properties (Variables)

This is the one people know but don't always use to its full potential:

```css
:root {
  /* Design tokens */
  --color-primary: oklch(55% 0.2 260);
  --color-primary-hover: oklch(45% 0.2 260);
  --color-surface: oklch(98% 0 0);
  --color-text: oklch(20% 0 0);

  --space-xs: 0.25rem;
  --space-sm: 0.5rem;
  --space-md: 1rem;
  --space-lg: 2rem;
  --space-xl: 4rem;

  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-full: 9999px;

  --font-body: system-ui, sans-serif;
  --font-mono: ui-monospace, "Cascadia Code", monospace;

  --shadow-sm: 0 1px 3px oklch(0% 0 0 / 0.1);
  --shadow-md: 0 4px 6px oklch(0% 0 0 / 0.1);
}
```

This looks like Sass variables. It isn't. These are live values in the DOM — they can be changed by JavaScript, they inherit through the document tree, they can be scoped to elements, and they respond to media queries.

```css
/* Scoped variables — only affect their subtree */
.card {
  --card-padding: var(--space-md);
  --card-radius: var(--radius-md);
}

.card.compact {
  --card-padding: var(--space-sm);
}

/* Use anywhere in the card's subtree */
.card-body {
  padding: var(--card-padding);
  border-radius: var(--card-radius);
}
```

Sass variables can't do this. They're compile-time constants. CSS custom properties are runtime values that participate in the cascade. That's a categorically different thing, and it enables patterns that Sass never could:

```javascript
// JavaScript can read and write CSS custom properties
const root = document.documentElement;

// Read the current value
const primary = getComputedStyle(root).getPropertyValue('--color-primary');

// Set a new theme dynamically
root.style.setProperty('--color-primary', 'oklch(55% 0.3 30)');
```

Theme switching without JavaScript class toggling, without injecting stylesheets, without recompilation. Just change the variable.

## CSS Nesting

CSS nesting shipped in all major browsers in 2023. It looks like this:

```css
.nav {
  display: flex;
  gap: var(--space-md);
  padding: var(--space-sm) var(--space-lg);
  background: var(--color-surface);

  & a {
    color: var(--color-text);
    text-decoration: none;
    padding: var(--space-xs) var(--space-sm);
    border-radius: var(--radius-sm);

    &:hover {
      background: oklch(from var(--color-primary) l c h / 0.1);
      color: var(--color-primary);
    }

    &[aria-current="page"] {
      font-weight: 600;
      color: var(--color-primary);
    }
  }

  @media (max-width: 768px) {
    flex-direction: column;

    & a {
      padding: var(--space-sm) var(--space-md);
    }
  }
}
```

The `&` refers to the parent selector. Media queries can be nested inside rules. This is Sass nesting syntax, largely. The difference: no compilation, no Sass. It's in the spec and in every modern browser.

One gotcha worth knowing: in CSS nesting, the `&` is required when nesting elements directly (unlike Sass where `& a` and `a` inside a rule are equivalent). The CSS Working Group made this explicit for parser ambiguity reasons. You need `& a`, not just `a`. Modern browsers handle both in most cases, but `&` is cleaner and more explicit.

## CSS `@layer` for Specificity Management

This is the feature Sass never had. `@layer` gives you explicit control over the specificity cascade, solving the most common source of "why isn't my CSS winning" — without `!important`.

```css
/* Define layer order — lower layers lose to higher layers */
@layer reset, base, components, utilities;

@layer reset {
  *, *::before, *::after {
    box-sizing: border-box;
  }
  body { margin: 0; }
}

@layer base {
  body {
    font-family: var(--font-body);
    color: var(--color-text);
    background: var(--color-surface);
  }

  h1, h2, h3, h4 {
    line-height: 1.2;
  }
}

@layer components {
  .button {
    display: inline-flex;
    align-items: center;
    padding: var(--space-sm) var(--space-md);
    background: var(--color-primary);
    color: white;
    border: none;
    border-radius: var(--radius-md);
    cursor: pointer;
    font-weight: 500;

    &:hover {
      background: var(--color-primary-hover);
    }
  }
}

@layer utilities {
  .visually-hidden {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    border: 0;
  }
}
```

Any rule in `utilities` wins over any rule in `components` regardless of specificity. A class selector in `utilities` beats an ID selector in `base`. The layer order defines the winner, not the selector weight.

This solves one of the oldest problems in CSS architecture: how to include a third-party stylesheet without its high-specificity selectors overriding your own. Wrap it in a layer:

```css
@layer third-party;

@import url('./vendor/some-library.css') layer(third-party);

/* Your styles are in a higher layer and always win */
@layer components {
  .button {
    /* This overrides .button from third-party, regardless of specificity */
    background: var(--color-primary);
  }
}
```

ITCSS, BEM, OOCSS, SMACSS — these methodologies existed to manage cascade order in the absence of `@layer`. With `@layer`, you don't need the methodology. You write the layer order declaration once and your cascade is explicitly controlled.

## `calc()`, `clamp()`, and Modern CSS Math

```css
:root {
  /* Fluid typography — scales between viewport sizes without media queries */
  --text-sm: clamp(0.875rem, 1vw + 0.5rem, 1rem);
  --text-base: clamp(1rem, 1.2vw + 0.5rem, 1.25rem);
  --text-lg: clamp(1.125rem, 1.5vw + 0.5rem, 1.5rem);
  --text-xl: clamp(1.5rem, 3vw + 0.5rem, 2.5rem);

  /* Fluid spacing */
  --space-section: clamp(3rem, 8vw, 8rem);
}
```

`clamp(min, preferred, max)` — the preferred value grows with the viewport, but is clamped between min and max. No JavaScript, no media query breakpoints for font size. The typography is fluid across all viewport widths.

```css
/* Complex layout calculations */
.sidebar-layout {
  display: grid;
  grid-template-columns: min(30%, 320px) 1fr;
  gap: var(--space-md);
}

/* Dynamic padding that accounts for scrollbar width */
.content {
  padding-inline: max(var(--space-md), calc((100vw - 1200px) / 2));
}
```

`min()`, `max()`, `clamp()`, `calc()` — these handle what Sass's `math.div()` and percentage calculations attempted to handle, and they do it at runtime with access to actual viewport dimensions.

## Container Queries: The Big One

Media queries respond to the viewport. Container queries respond to the element's container — which is what component-based design actually needs.

```css
/* Define a containment context */
.card-grid {
  container-type: inline-size;
  container-name: grid;

  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: var(--space-md);
}

/* Card styles based on its container, not the viewport */
.card {
  padding: var(--space-md);

  @container grid (min-width: 600px) {
    display: grid;
    grid-template-columns: auto 1fr;
    gap: var(--space-sm);
  }

  @container grid (min-width: 900px) {
    & .card-meta {
      display: flex;
      gap: var(--space-sm);
    }
  }
}
```

A card that's narrow (because it's in a narrow column) displays differently from a card that's wide (because it's in a wide column). This is not possible with media queries — media queries don't know anything about the card's container. This is the feature that makes proper responsive components possible without JavaScript measurement.

## Relative Colors and `oklch()`

Sass had `darken()`, `lighten()`, `saturate()`. CSS now has relative colors:

```css
:root {
  --brand: oklch(55% 0.2 260);

  /* Automatically derived variants */
  --brand-light: oklch(from var(--brand) calc(l + 0.15) c h);
  --brand-dark: oklch(from var(--brand) calc(l - 0.15) c h);
  --brand-muted: oklch(from var(--brand) l calc(c * 0.5) h);
  --brand-complement: oklch(from var(--brand) l c calc(h + 180));
}
```

`oklch()` is a perceptually uniform color space — colors that are numerically equidistant are also visually equidistant. Darken by 15% lightness and you get a consistently darker shade regardless of hue. Compare this to HSL where "darken by 15%" produces inconsistent results across different hues.

`color-mix()` for blending:

```css
.muted-text {
  color: color-mix(in oklch, var(--color-text) 50%, var(--color-surface));
}
```

Browser support: Chrome 111+, Firefox 113+, Safari 16.4+. If you're writing CSS today and targeting modern browsers, you have color functions that make Sass's color manipulation look like what it was — a workaround for a missing language feature.

## `@import` and File Organization

Sass's `@import` (and the newer `@use`/`@forward`) allowed splitting CSS into multiple files that got compiled into one. Native CSS `@import` works for this too, though with different performance characteristics:

```css
/* styles.css */
@import url('./reset.css');
@import url('./tokens.css');
@import url('./components/button.css');
@import url('./components/card.css');
@import url('./layout.css');
```

Each `@import` is a separate HTTP request. With HTTP/2, this is tolerable for small numbers of imports. For development, it's fine — you see the actual files in DevTools, source maps are your actual source, and there's no compilation delay.

For production, if you have many CSS files and care about the cascade of `@import` requests, concatenate them. This is simpler than a Sass compilation: `cat reset.css tokens.css components/*.css layout.css > styles.min.css`. One shell command, no build tool.

Or use PostCSS with only the `postcss-import` plugin — which processes `@import` and outputs concatenated CSS. No Sass, no preprocessor, just file concatenation.

## A Complete Design System Without Preprocessors

Here's a practical design system in plain CSS:

```css
/* tokens.css — Single source of truth for design decisions */
:root {
  /* Color palette in oklch for perceptual uniformity */
  --palette-blue-5: oklch(95% 0.05 260);
  --palette-blue-10: oklch(90% 0.08 260);
  --palette-blue-50: oklch(55% 0.2 260);
  --palette-blue-60: oklch(45% 0.2 260);
  --palette-blue-90: oklch(25% 0.15 260);

  --palette-neutral-0: oklch(100% 0 0);
  --palette-neutral-5: oklch(97% 0 0);
  --palette-neutral-10: oklch(93% 0 0);
  --palette-neutral-30: oklch(75% 0 0);
  --palette-neutral-60: oklch(45% 0 0);
  --palette-neutral-95: oklch(15% 0 0);

  /* Semantic tokens — reference palette tokens */
  --color-brand: var(--palette-blue-50);
  --color-brand-hover: var(--palette-blue-60);
  --color-surface: var(--palette-neutral-0);
  --color-surface-raised: var(--palette-neutral-5);
  --color-surface-sunken: var(--palette-neutral-10);
  --color-border: var(--palette-neutral-10);
  --color-text: var(--palette-neutral-95);
  --color-text-muted: var(--palette-neutral-60);
  --color-text-on-brand: white;

  /* Dark mode — same tokens, different values */
  @media (prefers-color-scheme: dark) {
    --color-surface: var(--palette-neutral-95);
    --color-surface-raised: oklch(20% 0 0);
    --color-surface-sunken: oklch(12% 0 0);
    --color-border: oklch(25% 0 0);
    --color-text: var(--palette-neutral-5);
    --color-text-muted: var(--palette-neutral-30);
  }

  /* Typography */
  --font-sans: system-ui, -apple-system, sans-serif;
  --font-mono: ui-monospace, "Cascadia Code", "Fira Code", monospace;

  --text-xs: clamp(0.75rem, 0.8vw + 0.4rem, 0.875rem);
  --text-sm: clamp(0.875rem, 0.9vw + 0.45rem, 1rem);
  --text-base: clamp(1rem, 1.1vw + 0.5rem, 1.125rem);
  --text-lg: clamp(1.125rem, 1.4vw + 0.55rem, 1.375rem);
  --text-xl: clamp(1.25rem, 2vw + 0.6rem, 1.75rem);
  --text-2xl: clamp(1.5rem, 3vw + 0.5rem, 2.5rem);
  --text-3xl: clamp(2rem, 5vw + 0.5rem, 4rem);

  /* Spacing */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-6: 1.5rem;
  --space-8: 2rem;
  --space-12: 3rem;
  --space-16: 4rem;

  /* Radii */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --radius-full: 9999px;

  /* Shadows */
  --shadow-sm: 0 1px 2px oklch(0% 0 0 / 0.05);
  --shadow-md: 0 4px 6px oklch(0% 0 0 / 0.07), 0 2px 4px oklch(0% 0 0 / 0.06);
  --shadow-lg: 0 10px 15px oklch(0% 0 0 / 0.1), 0 4px 6px oklch(0% 0 0 / 0.05);
}
```

```css
/* components/button.css */
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: var(--space-2);
  padding: var(--space-2) var(--space-4);
  font-family: var(--font-sans);
  font-size: var(--text-sm);
  font-weight: 500;
  line-height: 1.25;
  border-radius: var(--radius-md);
  border: 1px solid transparent;
  cursor: pointer;
  text-decoration: none;
  transition: background-color 150ms, border-color 150ms, color 150ms;

  &:focus-visible {
    outline: 2px solid var(--color-brand);
    outline-offset: 2px;
  }

  /* Variants */
  &.btn-primary {
    background: var(--color-brand);
    color: var(--color-text-on-brand);

    &:hover { background: var(--color-brand-hover); }
  }

  &.btn-secondary {
    background: var(--color-surface);
    color: var(--color-text);
    border-color: var(--color-border);

    &:hover {
      background: var(--color-surface-sunken);
      border-color: var(--color-text-muted);
    }
  }

  &.btn-ghost {
    background: transparent;
    color: var(--color-text);

    &:hover { background: var(--color-surface-sunken); }
  }

  /* Sizes */
  &.btn-sm {
    padding: var(--space-1) var(--space-3);
    font-size: var(--text-xs);
  }

  &.btn-lg {
    padding: var(--space-3) var(--space-6);
    font-size: var(--text-lg);
  }

  &[disabled], &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
}
```

This design system has dark mode, fluid typography, semantic color tokens, full button variants with all interactive states, and no preprocessing. It runs in the browser as-is.

## What Sass Still Does

Let's be honest about what you lose:

**`@mixin` and `@include`.** Sass mixins allow parameterized blocks of CSS. CSS doesn't have these. You can approximate them with custom properties (pass the parameter as a variable), but it's not the same.

**`@each` and `@for` loops.** Generating CSS programmatically. CSS doesn't have loops. For cases where you're generating utility classes (`mt-1` through `mt-16`), Sass still wins.

**`@extend`.** Sharing rule sets without repetition. No native CSS equivalent.

**Complex functions.** Sass lets you write functions that compute values at compile time. CSS `calc()` is powerful but operates at runtime.

If your workflow relies heavily on generated utility classes (Tailwind-style), you're still going to want a preprocessor or PostCSS at minimum. If you're writing component styles with a design token system, you probably don't need one.

---

The CSS of 2024 is not the CSS of 2006. The features that justified Sass — variables, nesting, calculation, color manipulation — are now in the language. The features that Sass still does better (loops, mixins, programmatic generation) matter for certain architectures and not for others.

The default answer used to be "yes, use Sass." The honest answer now is "it depends, and the bar for needing Sass is higher than it used to be." That's progress.
