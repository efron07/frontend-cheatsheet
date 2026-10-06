# 11. Tailwind CSS

## 11.1 What Tailwind is

**What it is:** Tailwind is a **utility-first** CSS framework. Instead of writing CSS classes like `.medicine-card` in a separate file, you style elements directly with small single-purpose classes: `p-4` (padding), `rounded-lg`, `bg-white`, `text-sm`. Tailwind scans your files and generates only the CSS for the classes you actually use.

```html
<!-- Traditional CSS: invent a name, write CSS elsewhere -->
<div class="medicine-card">...</div>

<!-- Tailwind: style in place -->
<div class="rounded-lg border border-gray-200 bg-white p-4 shadow-sm hover:shadow-md">...</div>
```

**Why it matters:** no naming, no dead CSS, no jumping between files, a consistent design scale (spacing, colours, sizes) built in, and tiny production CSS. It is the default styling choice in Next.js projects.

## 11.2 Setup (Tailwind v4)

Tailwind v4 is configured **in CSS**; there is no `tailwind.config.js` by default.

**Next.js** (selected automatically by `create-next-app` if you choose Tailwind):

```bash
npm install tailwindcss @tailwindcss/postcss postcss
```

```js
// postcss.config.mjs
export default { plugins: { '@tailwindcss/postcss': {} } };
```

```css
/* app/globals.css */
@import 'tailwindcss';
```

**Vite + React:**

```bash
npm install tailwindcss @tailwindcss/vite
```

```ts
// vite.config.ts
import tailwindcss from '@tailwindcss/vite';
export default defineConfig({ plugins: [react(), tailwindcss()] });
```

**Editor support:** install the **Tailwind CSS IntelliSense** VS Code extension (autocomplete, hover previews) and the Prettier plugin `prettier-plugin-tailwindcss` (sorts classes in a consistent order).

## 11.3 The core utilities

**Spacing scale:** `1` = 0.25rem (4px). So `p-4` = 16px, `m-2` = 8px, `gap-6` = 24px.

| Area | Classes | Meaning |
| --- | --- | --- |
| Padding | `p-4`, `px-4`, `py-2`, `pt-1`, `ps-3` | All sides / x / y / top / inline-start |
| Margin | `m-4`, `mx-auto`, `mt-6`, `-mt-2` | `mx-auto` centres a block |
| Width / height | `w-full`, `w-64`, `max-w-4xl`, `h-screen`, `min-h-dvh`, `size-10` | `size-*` sets width and height |
| Display | `block`, `inline-flex`, `grid`, `hidden` |  |
| Flex | `flex`, `flex-col`, `items-center`, `justify-between`, `gap-4`, `flex-1`, `shrink-0`, `flex-wrap` |  |
| Grid | `grid`, `grid-cols-3`, `col-span-2`, `grid-cols-[1fr_380px]` | Brackets = arbitrary value |
| Position | `relative`, `absolute`, `fixed`, `sticky`, `top-0`, `inset-0`, `z-10` |  |
| Typography | `text-sm`, `text-2xl`, `font-semibold`, `leading-6`, `tracking-tight`, `text-center`, `truncate`, `line-clamp-2` |  |
| Colour | `text-gray-700`, `bg-red-600`, `border-gray-200`, `bg-red-600/10` | `/10` = 10% opacity |
| Borders | `border`, `border-2`, `rounded-md`, `rounded-full`, `divide-y` |  |
| Effects | `shadow-sm`, `opacity-50`, `ring-2 ring-red-500` |  |
| Transitions | `transition`, `duration-150`, `ease-out`, `animate-spin`, `animate-pulse` |  |
| Overflow | `overflow-auto`, `overflow-x-auto`, `overflow-hidden` |  |

**Arbitrary values** when the scale lacks what you need: `w-[340px]`, `bg-[#0f766e]`, `grid-cols-[1fr_340px]`, `text-[13px]`.

## 11.4 States and variants

**What it is:** Prefix a utility to apply it only in a state.

```html
<button class="bg-red-600 text-white hover:bg-red-700 focus-visible:outline-2 focus-visible:outline-offset-2
               focus-visible:outline-red-600 active:scale-95 disabled:cursor-not-allowed disabled:opacity-50">
  Complete sale
</button>

<input class="border invalid:border-red-500 focus:border-blue-500 placeholder:text-gray-400" />

<!-- group: style children when the parent is hovered -->
<a class="group block rounded-lg p-4 hover:bg-gray-50">
  <h3 class="font-medium group-hover:text-red-600">Amoxicillin</h3>
</a>

<!-- peer: style a sibling based on another sibling's state -->
<input type="checkbox" class="peer" id="rx" />
<label for="rx" class="peer-checked:font-bold">Prescription verified</label>

<!-- data and aria attributes -->
<div data-state="open" class="data-[state=open]:block hidden">...</div>
<button aria-expanded="true" class="aria-expanded:bg-gray-100">Menu</button>
```

Other variants: `first:`, `last:`, `odd:`, `even:`, `has-[input:checked]:`, `print:hidden`, `motion-reduce:transition-none`.

## 11.5 Responsive design

Tailwind is **mobile-first**: unprefixed classes apply to all sizes; prefixes apply **from that width up**.

| Prefix | Min width |
| --- | --- |
| `sm:` | 640px |
| `md:` | 768px |
| `lg:` | 1024px |
| `xl:` | 1280px |
| `2xl:` | 1536px |

```html
<!-- 1 column on phones, 2 on tablets, 4 on desktops -->
<div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">...</div>

<!-- POS: stacked on mobile, products + cart side by side on desktop -->
<div class="flex flex-col lg:grid lg:grid-cols-[1fr_380px] lg:h-dvh">...</div>

<!-- Hide on mobile, show from md -->
<aside class="hidden md:block">...</aside>

<!-- Container queries (built into v4) -->
<div class="@container">
  <div class="flex flex-col @md:flex-row">...</div>
</div>
```

## 11.6 Dark mode

By default, `dark:` follows the operating system setting:

```html
<div class="bg-white text-gray-900 dark:bg-gray-900 dark:text-gray-100">...</div>
```

To control it with a toggle (a `dark` class on `<html>`), add in your CSS:

```css
@custom-variant dark (&:where(.dark, .dark *));
```

## 11.7 Theming: your brand in `@theme`

**What it is:** In v4 you define design tokens as CSS variables inside `@theme`. Each one creates matching utilities automatically.

```css
/* app/globals.css */
@import 'tailwindcss';

@theme {
  --color-brand-50: oklch(0.97 0.02 155);
  --color-brand-500: oklch(0.62 0.15 155);     /* PharmaCare green */
  --color-brand-600: oklch(0.55 0.15 155);
  --color-brand-700: oklch(0.47 0.13 155);
  --font-sans: 'Inter', ui-sans-serif, system-ui, sans-serif;
  --breakpoint-3xl: 1920px;
  --radius-card: 0.75rem;
}
```

```html
<button class="bg-brand-600 hover:bg-brand-700 rounded-card font-sans 3xl:text-lg">Pay</button>
```

The variables are also usable in plain CSS: `color: var(--color-brand-600);`.

**Custom utilities and component classes:**

```css
@utility scrollbar-hidden {
  scrollbar-width: none;
}

/* @apply reuses utilities inside CSS — use sparingly; prefer React components */
@layer components {
  .btn-primary { @apply rounded-md bg-brand-600 px-4 py-2 font-medium text-white hover:bg-brand-700; }
}

@plugin '@tailwindcss/typography';   /* plugins: prose styles for articles */
@source '../packages/ui';            /* scan an extra folder for class names */
```

## 11.8 Tailwind in React: components and the `cn()` helper

**The right way to avoid repeated class lists is a React component**, not `@apply`.

```bash
npm install clsx tailwind-merge
```

```ts
// lib/utils.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

// clsx: join classes conditionally. twMerge: resolve conflicts (p-2 + p-4 → p-4)
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

```tsx
// components/ui/stock-badge.tsx
import { cn } from '@/lib/utils';

export function StockBadge({ stock, className }: { stock: number; className?: string }) {
  return (
    <span
      className={cn(
        'inline-flex items-center rounded-full px-2 py-0.5 text-xs font-medium',
        stock === 0 && 'bg-red-100 text-red-700',
        stock > 0 && stock < 20 && 'bg-amber-100 text-amber-800',
        stock >= 20 && 'bg-green-100 text-green-700',
        className,                       // caller can override
      )}
    >
      {stock === 0 ? 'Out of stock' : `${stock} in stock`}
    </span>
  );
}
```

**Variants with `cva`** (class-variance-authority, used by shadcn/ui):

```ts
import { cva, type VariantProps } from 'class-variance-authority';

export const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md font-medium transition disabled:opacity-50',
  {
    variants: {
      variant: { primary: 'bg-brand-600 text-white hover:bg-brand-700', outline: 'border hover:bg-gray-50', danger: 'bg-red-600 text-white' },
      size: { sm: 'h-8 px-3 text-sm', md: 'h-10 px-4', lg: 'h-12 px-6 text-lg' },
    },
    defaultVariants: { variant: 'primary', size: 'md' },
  },
);
export type ButtonVariants = VariantProps<typeof buttonVariants>;
```

**Rule: never build class names dynamically.** Tailwind finds classes by scanning your text, so `` `bg-${color}-600` `` is never generated. Map to full class names instead:

```ts
const STATUS_CLASSES = { paid: 'bg-green-100 text-green-700', voided: 'bg-red-100 text-red-700' } as const;
```

## 11.9 A complete PharmaCare card

```tsx
export function MedicineCard({ medicine }: { medicine: Medicine }) {
  return (
    <article className="group flex flex-col overflow-hidden rounded-xl border border-gray-200 bg-white shadow-sm transition hover:shadow-md dark:border-gray-800 dark:bg-gray-900">
      <div className="aspect-[4/3] bg-gray-100" />
      <div className="flex flex-1 flex-col gap-2 p-4">
        <div className="flex items-start justify-between gap-2">
          <h3 className="line-clamp-2 font-semibold text-gray-900 group-hover:text-brand-700 dark:text-gray-100">
            {medicine.name}
          </h3>
          {medicine.requiresPrescription && (
            <span className="shrink-0 rounded bg-blue-100 px-1.5 text-xs font-bold text-blue-700">Rx</span>
          )}
        </div>
        <p className="text-sm text-gray-500">{medicine.category.name}</p>
        <div className="mt-auto flex items-center justify-between pt-2">
          <span className="text-lg font-bold">{medicine.price.toLocaleString()} TZS</span>
          <StockBadge stock={medicine.stock} />
        </div>
      </div>
    </article>
  );
}
```

**Component libraries built on Tailwind:** shadcn/ui (copy-in components on Radix, section 16), Headless UI, daisyUI, Flowbite.
