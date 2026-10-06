# 10. HTML and CSS essentials

React components output HTML and Tailwind writes CSS, so you need the fundamentals of both.

## 10.1 Semantic HTML

**What it is:** Using the HTML element that describes the **meaning** of the content (`<nav>`, `<button>`, `<table>`) instead of generic `<div>`s. Screen readers, search engines and browsers all understand it better.

```html
<header>
  <nav aria-label="Main">
    <a href="/">PharmaCare</a>
    <a href="/medicines">Medicines</a>
  </nav>
</header>

<main>
  <h1>Medicines</h1>                         <!-- one h1 per page; headings in order h1 > h2 > h3 -->
  <section aria-labelledby="painkillers">
    <h2 id="painkillers">Painkillers</h2>
    <article>
      <h3>Paracetamol 500mg</h3>
      <img src="/img/paracetamol.webp" alt="Box of Paracetamol 500mg tablets" width="400" height="300" />
      <p>Price: <data value="300">300 TZS</data></p>
      <button type="button">Add to cart</button>
    </article>
  </section>

  <table>
    <caption>Today's sales</caption>
    <thead><tr><th scope="col">Receipt</th><th scope="col">Total</th></tr></thead>
    <tbody><tr><td>RCP-00012</td><td>12,000</td></tr></tbody>
  </table>
</main>

<footer>
  <address>Dodoma, Tanzania</address>
</footer>
```

| Use | Not |
| --- | --- |
| `<button>` for actions | `<div onclick>` (not keyboard accessible) |
| `<a href>` for navigation | `<span onclick="location=...">` |
| `<ul>`/`<ol>` for lists | A stack of `<div>`s |
| `<table>` for tabular data | Grids of divs for data |
| `<label for>` with every input | Placeholder as the only label |

## 10.2 Forms in HTML

```html
<form action="/batches" method="post">
  <label for="batchNo">Batch number</label>
  <input id="batchNo" name="batchNo" required pattern="[A-Z]{3}-\d{4}-\d{2}" autocomplete="off" />

  <label for="qty">Quantity</label>
  <input id="qty" name="quantity" type="number" min="1" inputmode="numeric" required />

  <label for="expiry">Expiry date</label>
  <input id="expiry" name="expiresAt" type="date" required />

  <fieldset>
    <legend>Payment</legend>
    <label><input type="radio" name="method" value="cash" checked /> Cash</label>
    <label><input type="radio" name="method" value="mpesa" /> M-Pesa</label>
  </fieldset>

  <select name="category"><option value="1">Antibiotics</option></select>
  <textarea name="notes" rows="3"></textarea>
  <button type="submit">Save</button>
</form>
```

Useful input types: `text`, `email`, `tel`, `number`, `date`, `search`, `password`, `file`, `checkbox`, `radio`. `inputmode` picks the phone keyboard (numeric keypad for quantities).

## 10.3 Accessibility basics (a11y)

**What it is:** Making the app usable by everyone, including people using keyboards, screen readers or zoom. It is also a legal requirement in many markets and helps SEO.

- Every image has meaningful `alt` text (or `alt=""` if decorative).
- Every input has a visible `<label>`.
- Everything clickable works with the keyboard (Tab, Enter, Space) and shows a visible focus ring.
- Colour contrast at least 4.5:1 for normal text. Never use colour alone ("red = expired"): add text or an icon.
- Use ARIA only when no native element exists: `aria-label`, `aria-expanded`, `aria-live="polite"` for toast messages, `role="dialog"`.

More in section 26.

## 10.4 CSS fundamentals

**What it is:** CSS (Cascading Style Sheets) controls how HTML looks. A rule = a **selector** + **declarations**.

```css
.medicine-card {             /* class selector */
  padding: 16px;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
}
#total { font-weight: 700; }                  /* id selector */
button:hover { background: #b91c1c; }         /* pseudo-class */
.card > h3 { margin: 0; }                     /* direct child */
input[type='number'] { width: 6rem; }         /* attribute selector */
.badge::before { content: '● '; }            /* pseudo-element */
```

**Cascade and specificity** — when two rules conflict, the more specific one wins: inline styles > `#id` > `.class` / `:hover` / `[attr]` > `element`. If equal, the later rule wins. `!important` overrides everything (avoid).

**Units:** `px` (fixed), `rem` (relative to the root font size, usually 16px — use for sizing and spacing), `em` (relative to the parent), `%`, `vw`/`vh` (viewport), `dvh` (dynamic viewport height on mobile), `fr` (grid fraction).

## 10.5 The box model

Every element is a box: **content** → **padding** (inside space) → **border** → **margin** (outside space).

```css
*, *::before, *::after { box-sizing: border-box; }   /* width includes padding and border — always set this */
```

**Display types:** `block` (full width, new line: `div`, `p`), `inline` (flows in text: `span`, `a`), `inline-block`, `flex`, `grid`, `none`.

**Position:** `static` (default), `relative` (offset from normal place; anchor for absolute children), `absolute` (placed against the nearest positioned parent), `fixed` (against the viewport — a fixed POS footer), `sticky` (sticks while scrolling — table headers).

## 10.6 Flexbox — one-dimensional layout

**What it is:** Lays children out in a **row or a column**, with control over alignment and spacing.

```css
.toolbar {
  display: flex;
  flex-direction: row;          /* or column */
  justify-content: space-between;  /* main axis: start | center | end | space-between */
  align-items: center;          /* cross axis */
  gap: 12px;
  flex-wrap: wrap;
}
.search { flex: 1; }            /* take the remaining space */
```

## 10.7 Grid — two-dimensional layout

**What it is:** Rows **and** columns at the same time. Best for page layouts and card grids.

```css
.medicine-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));  /* responsive cards, no media query */
  gap: 16px;
}

.pos-layout {
  display: grid;
  grid-template-columns: 1fr 380px;      /* products | cart */
  grid-template-rows: auto 1fr;
  height: 100dvh;
}
```

## 10.8 Responsive design

**What it is:** One layout that adapts to phones, tablets and desktops.

```html
<meta name="viewport" content="width=device-width, initial-scale=1" />   <!-- required -->
```

```css
/* Mobile-first: base styles for small screens, then add for bigger ones */
.pos-layout { display: block; }

@media (min-width: 768px) {        /* tablet and up */
  .pos-layout { display: grid; grid-template-columns: 1fr 340px; }
}

/* Container queries: respond to the size of the parent, not the screen */
.card-wrapper { container-type: inline-size; }
@container (min-width: 400px) { .card { display: flex; } }
```

## 10.9 Modern CSS worth knowing

- **Custom properties (CSS variables):** `:root { --brand: #dc2626; } .btn { background: var(--brand); }` — the base of theming and dark mode.
- **Native nesting:** `.card { & h3 { margin: 0; } &:hover { ... } }`.
- `clamp()` for fluid type: `font-size: clamp(1rem, 2.5vw, 1.5rem)`.
- `:has()` parent selector: `.field:has(input:invalid) { border-color: red; }`.
- Transitions and animations: `transition: transform 150ms ease;`, `@keyframes`.
- `@layer` controls cascade order (Tailwind uses it).

**Ways to write CSS in React projects:** plain global CSS, **CSS Modules** (`Card.module.css`, class names scoped per component), **Sass** (variables, nesting, mixins — `npm i -D sass`), **CSS-in-JS** (styled-components, Emotion — runtime cost, poor fit for Server Components), zero-runtime libraries (Panda CSS, vanilla-extract), and **Tailwind CSS** (next section). Tailwind is the default choice for Next.js today.
