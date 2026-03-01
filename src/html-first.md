# HTML-First Development

*What You Get for Free*

---

The history of web development is a history of forgetting what HTML already does.

Every few years, the frontend community rediscovers something the browser has handled natively — and then builds a library around it. Form validation. Dialog elements. Accordion components. Popovers. The carousel-of-the-week on npm, which wraps `<details>` in 50KB of JavaScript because nobody checked whether `<details>` would suffice.

`<details>` frequently suffices.

This chapter is about reading the HTML spec before reaching for JavaScript, and what you find when you do.

## The Interactive Elements You Already Have

### Details and Summary: Free Accordion

```html
<details>
  <summary>What is the zero build movement?</summary>
  <p>A philosophy of using native browser capabilities rather than build
  tools wherever possible — ES modules, import maps, modern CSS, and
  semantic HTML — instead of adding compilation steps to solve problems
  the platform already handles.</p>
</details>
```

This renders as a clickable disclosure widget. The triangle indicator is browser-provided. The open/close animation can be customized with CSS. No JavaScript. No library. The `open` attribute controls the initial state:

```html
<details open>
  <summary>Expanded by default</summary>
  <p>This one starts open.</p>
</details>
```

For CSS styling:

```css
details {
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  padding: var(--space-4);
}

details[open] summary {
  margin-bottom: var(--space-3);
  border-bottom: 1px solid var(--color-border);
  padding-bottom: var(--space-3);
}

summary {
  cursor: pointer;
  font-weight: 600;
  list-style: none; /* Remove default triangle */
}

summary::after {
  content: '+';
  float: right;
}

details[open] summary::after {
  content: '−';
}
```

FAQs, accordions, "show more" sections — `<details>` handles all of these.

### Dialog: The Modal Element

The `<dialog>` element has been in browsers since 2022 (and Chrome/Opera since 2014). It's a proper modal with:

- Focus trapping (keyboard navigation stays within the dialog while it's open)
- The `::backdrop` pseudo-element for the overlay
- `Escape` key to close
- `show()` / `showModal()` / `close()` methods
- The `open` attribute

```html
<button id="open-btn" type="button">Open Dialog</button>

<dialog id="my-dialog">
  <h2>Confirm Action</h2>
  <p>Are you sure you want to proceed?</p>
  <menu>
    <li><button id="confirm-btn" type="button">Confirm</button></li>
    <li><button id="cancel-btn" type="button">Cancel</button></li>
  </menu>
</dialog>
```

```javascript
const dialog = document.getElementById('my-dialog');

document.getElementById('open-btn').addEventListener('click', () => {
  dialog.showModal(); // Opens as modal with focus trap and backdrop
});

document.getElementById('cancel-btn').addEventListener('click', () => {
  dialog.close();
});

document.getElementById('confirm-btn').addEventListener('click', () => {
  // Do the thing
  dialog.close('confirmed'); // Can pass a return value
});

// The dialog fires a 'close' event, has a returnValue property
dialog.addEventListener('close', () => {
  if (dialog.returnValue === 'confirmed') {
    console.log('User confirmed');
  }
});
```

```css
dialog {
  border: 1px solid var(--color-border);
  border-radius: var(--radius-lg);
  padding: var(--space-8);
  max-width: 500px;
  width: 90%;
  box-shadow: var(--shadow-lg);
}

dialog::backdrop {
  background: oklch(0% 0 0 / 0.5);
  backdrop-filter: blur(4px);
}
```

This is a proper, accessible modal dialog — focus trap, escape-to-close, backdrop. Every "modal component" npm library is wrapping something that does less than this.

### Popover API: Tooltips and Dropdowns Without JavaScript

The Popover API shipped in all major browsers in 2023. It's a mechanism for showing overlaid content — tooltips, dropdowns, command palettes — with none of the JavaScript positioning code:

```html
<button popovertarget="user-menu">Account</button>

<menu id="user-menu" popover>
  <li><a href="/profile">Profile</a></li>
  <li><a href="/settings">Settings</a></li>
  <li><button type="button">Sign out</button></li>
</menu>
```

That's it. The button with `popovertarget` opens and closes the element with `popover`. No JavaScript. The popover:

- Appears in the top layer (above everything, including modals)
- Dismisses on Escape
- Dismisses on click outside (light dismiss)
- Is accessible with proper ARIA behavior

For positioning, the CSS Anchor Positioning API (shipping 2024) positions the popover relative to the button:

```css
#user-menu {
  position-anchor: --user-menu-anchor;
  position-area: bottom span-right;
  margin-top: var(--space-1);
}

[popovertarget="user-menu"] {
  anchor-name: --user-menu-anchor;
}
```

Anchor positioning is Chrome 125+; Firefox and Safari are catching up. For now, you can supplement with a small JavaScript positioning helper. But the structure — the open/close behavior, the top-layer stacking, the light dismiss — is free.

## Native Form Validation

HTML5 form validation has been available for over a decade and is wildly underused:

```html
<form id="signup-form">
  <fieldset>
    <legend>Create account</legend>

    <label for="email">Email</label>
    <input
      type="email"
      id="email"
      name="email"
      required
      autocomplete="email"
    >

    <label for="password">Password</label>
    <input
      type="password"
      id="password"
      name="password"
      required
      minlength="8"
      pattern="(?=.*[A-Z])(?=.*[0-9]).{8,}"
      aria-describedby="password-hint"
    >
    <p id="password-hint">At least 8 characters, one uppercase, one number.</p>

    <label for="confirm">Confirm password</label>
    <input
      type="password"
      id="confirm"
      name="confirm"
      required
    >

    <button type="submit">Create account</button>
  </fieldset>
</form>
```

```javascript
const form = document.getElementById('signup-form');

// Only intercept to add custom cross-field validation
form.addEventListener('submit', (e) => {
  const password = form.elements.password.value;
  const confirm = form.elements.confirm.value;

  if (password !== confirm) {
    form.elements.confirm.setCustomValidity("Passwords don't match");
    form.elements.confirm.reportValidity();
    e.preventDefault();
    return;
  }

  form.elements.confirm.setCustomValidity(''); // Clear error
  // Form is valid — submit or handle with fetch
});
```

CSS styling of validation states:

```css
input:invalid:not(:placeholder-shown) {
  border-color: var(--color-error);
}

input:valid:not(:placeholder-shown) {
  border-color: var(--color-success);
}

input:invalid:focus {
  outline-color: var(--color-error);
}
```

The `:not(:placeholder-shown)` trick avoids showing validation errors on empty, untouched fields. This gives you "validate on change after first interaction" without JavaScript.

`setCustomValidity()` integrates your custom errors into the browser's native validation bubbles. `reportValidity()` triggers display of those bubbles. You get accessible error messaging that announces via screen readers, positioned relative to the input, without writing any accessibility code yourself.

## Web Components: Custom Elements That Actually Work

Web components are four APIs that work together:

- **Custom Elements**: Define new HTML elements with JavaScript behavior
- **Shadow DOM**: Encapsulated DOM tree with scoped CSS
- **HTML Templates**: Inert, parseable markup for creating instances
- **Declarative Shadow DOM**: Server-rendered shadow DOM without JavaScript

Custom Elements without Shadow DOM are straightforward:

```javascript
// Define a reusable component
class UserAvatar extends HTMLElement {
  static get observedAttributes() {
    return ['name', 'size', 'src'];
  }

  connectedCallback() {
    this.render();
  }

  attributeChangedCallback() {
    this.render();
  }

  render() {
    const name = this.getAttribute('name') ?? 'User';
    const size = this.getAttribute('size') ?? '40';
    const src = this.getAttribute('src');
    const initials = name.split(' ').map(n => n[0]).join('').slice(0, 2);

    if (src) {
      this.innerHTML = `
        <img src="${src}"
             alt="${name}"
             width="${size}"
             height="${size}"
             style="border-radius:50%;width:${size}px;height:${size}px">
      `;
    } else {
      this.innerHTML = `
        <div style="
          width:${size}px;height:${size}px;
          border-radius:50%;
          background:var(--color-brand);
          color:white;
          display:flex;align-items:center;justify-content:center;
          font-size:${Number(size) * 0.4}px;font-weight:600;
        ">${initials}</div>
      `;
    }
  }
}

customElements.define('user-avatar', UserAvatar);
```

```html
<!-- Use it like any HTML element -->
<user-avatar name="Alice Chen" size="48"></user-avatar>
<user-avatar name="Bob Smith" src="/avatars/bob.jpg" size="32"></user-avatar>
```

This element is observable, updates when attributes change, works with innerHTML, document.createElement, and server-rendered HTML. No framework required.

With Shadow DOM:

```javascript
class ToastMessage extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    const { type = 'info', message } = this.dataset;

    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          padding: 1rem 1.5rem;
          border-radius: 8px;
          font-family: system-ui, sans-serif;
        }
        :host([data-type="error"]) { background: #fee; border: 1px solid #fcc; }
        :host([data-type="success"]) { background: #efe; border: 1px solid #cfc; }
        :host([data-type="info"]) { background: #eef; border: 1px solid #ccf; }
        button { float: right; background: none; border: none; cursor: pointer; }
      </style>
      <button aria-label="Dismiss">×</button>
      <slot></slot>
    `;

    this.shadowRoot.querySelector('button').addEventListener('click', () => {
      this.remove();
    });
  }
}

customElements.define('toast-message', ToastMessage);
```

```html
<toast-message data-type="success">
  Your changes have been saved.
</toast-message>
```

The CSS in the Shadow DOM is fully encapsulated — no leakage in or out. The `:host` pseudo-class styles the element itself from within its shadow root. `<slot>` is where light DOM children appear.

## Declarative Shadow DOM: SSR Web Components

Declarative Shadow DOM lets you render shadow DOM from the server, without JavaScript:

```html
<user-card>
  <template shadowrootmode="open">
    <style>
      :host { display: flex; align-items: center; gap: 1rem; }
      .info h3 { margin: 0; }
      .info p { margin: 0; color: #666; }
    </style>
    <slot name="avatar"></slot>
    <div class="info">
      <slot name="name"></slot>
      <slot name="role"></slot>
    </div>
  </template>
  <img slot="avatar" src="/avatar.jpg" alt="Alice" width="48" height="48">
  <h3 slot="name">Alice Chen</h3>
  <p slot="role">Senior Engineer</p>
</user-card>
```

This renders in the browser with an encapsulated shadow DOM, no JavaScript required. If JavaScript loads and a custom element class is registered for `user-card`, it can add behavior without disrupting the existing rendering. If JavaScript doesn't load, the HTML renders correctly on its own.

This is the right way to think about web components: progressive enhancement at the component level. The HTML always works. JavaScript adds behavior.

## `<template>` for Reusable Markup

The `<template>` element holds HTML that isn't rendered but can be cloned and inserted:

```html
<template id="card-template">
  <article class="card">
    <header>
      <h3 class="card-title"></h3>
      <span class="card-badge"></span>
    </header>
    <div class="card-body"></div>
    <footer class="card-footer">
      <button class="card-action" type="button">View details</button>
    </footer>
  </article>
</template>
```

```javascript
function createCard({ title, badge, body, onView }) {
  const template = document.getElementById('card-template');
  const clone = template.content.cloneNode(true);

  clone.querySelector('.card-title').textContent = title;
  clone.querySelector('.card-badge').textContent = badge;
  clone.querySelector('.card-body').textContent = body;
  clone.querySelector('.card-action').addEventListener('click', onView);

  return clone;
}

// Use it
const card = createCard({
  title: 'Server status',
  badge: 'Healthy',
  body: 'All systems operational.',
  onView: () => navigate('/status'),
});
document.getElementById('dashboard').appendChild(card);
```

This is a render function with no framework. It clones the template (which the browser has already parsed), fills in values, attaches events, and returns a ready-to-insert DOM fragment. Fast, explicit, and debuggable.

## Input Types That Aren't Text

A significant fraction of custom date pickers, color pickers, range sliders, and file uploads exist because developers didn't know the input type for these things exists:

```html
<!-- Date picker -->
<input type="date" min="2024-01-01" max="2024-12-31">

<!-- Date and time -->
<input type="datetime-local">

<!-- Month picker -->
<input type="month">

<!-- Color picker -->
<input type="color" value="#3b82f6">

<!-- Range slider -->
<input type="range" min="0" max="100" step="5" value="50">

<!-- File upload -->
<input type="file" accept="image/*" multiple>

<!-- Search with clear button -->
<input type="search" placeholder="Search...">
```

`type="date"` gives you a native date picker. It looks different in different browsers and OSes, which is either a feature (it matches what users expect on their platform) or a limitation (it doesn't match your design system). For internal tools: feature. For consumer apps where brand consistency matters: maybe a custom component is warranted.

## What HTML-First Buys You

**Accessibility by default.** Native HTML elements have ARIA semantics built in. `<button>` is keyboard-navigatable and activatable without JavaScript. `<dialog>` has focus management. `<input type="email">` announces its purpose to screen readers. When you replace native elements with custom JavaScript widgets, you're taking on the responsibility of implementing all of this yourself — and most custom implementations miss something.

**Performance.** Parsing HTML is one of the fastest things browsers do. A `<details>` element toggling open adds no render cost. A JavaScript-powered accordion has initialization cost, event handler cost, and potential layout thrash.

**Progressive enhancement.** HTML works before JavaScript loads. A form with native validation works even if your validation library fails. A `<dialog>` can have its basic behavior provided by HTML and its enhanced behavior added by JavaScript — without either being required for the other.

**Less JavaScript means less breakage.** JavaScript can fail. CDNs go down. Network requests fail. Batteries die mid-load. HTML doesn't have these problems. Every line of JavaScript you replace with semantic HTML is a line that works in degraded conditions.

---

The question to ask before reaching for a component library isn't "which library handles this?" It's "does the browser already handle this?" The answer is yes more often than the ecosystem assumes.

Web components, the Popover API, `<dialog>`, `<details>`, native form validation — these aren't replacements for every UI library in every case. They're the floor. Know the floor before you build on top of it.
