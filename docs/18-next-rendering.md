# 18. Server and Client Components, rendering strategies

## 18.1 Server Components vs Client Components

**What it is:** In the App Router, every component is a **Server Component** by default. It runs only on the server, can `await` data directly from a database or API, and sends finished HTML to the browser — **with zero JavaScript** for that component. A **Client Component** (marked with `'use client'`) is also pre-rendered to HTML on the server, but its JS is sent to the browser so it can use state, effects and event handlers.

| Need | Server Component | Client Component |
| --- | --- | --- |
| Fetch data, query the database, read secrets | Yes | No (secrets would leak) |
| `async`/`await` in the component | Yes | No |
| `useState`, `useEffect`, `useReducer`, context | No | Yes |
| `onClick`, `onChange`, other events | No | Yes |
| Browser APIs (`window`, `localStorage`) | No | Yes |
| Adds JS to the bundle | No | Yes |

**Rule: keep components on the server by default; add `'use client'` only to the small interactive leaves** (buttons, forms, search box, cart).

```tsx
// app/medicines/[slug]/page.tsx — SERVER component
import { AddToCartButton } from './add-to-cart-button';

export default async function MedicinePage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const medicine = await db.medicine.findUnique({ where: { slug }, include: { category: true } });  // direct DB access
  if (!medicine) notFound();

  return (
    <article className="mx-auto max-w-3xl p-6">
      <h1 className="text-3xl font-bold">{medicine.name}</h1>
      <p className="text-gray-600">{medicine.category.name}</p>
      <p className="mt-4 text-2xl">{medicine.price.toLocaleString()} TZS</p>
      <AddToCartButton medicine={medicine} />      {/* only this part ships JS */}
    </article>
  );
}
```

```tsx
// app/medicines/[slug]/add-to-cart-button.tsx — CLIENT component
'use client';
import { useCartStore } from '@/stores/cart-store';

export function AddToCartButton({ medicine }: { medicine: Medicine }) {
  const add = useCartStore((s) => s.add);
  return (
    <button onClick={() => add(medicine)} disabled={medicine.stock === 0}
      className="mt-6 rounded-md bg-brand-600 px-5 py-3 text-white disabled:opacity-50">
      {medicine.stock === 0 ? 'Out of stock' : 'Add to cart'}
    </button>
  );
}
```

## 18.2 Composition rules between server and client

1. **`'use client'` marks a boundary.** Everything imported into a Client Component file becomes client code too. Put the directive as low in the tree as possible.
2. **Server → Client props must be serialisable:** strings, numbers, booleans, plain objects, arrays, Dates, Promises. Not functions or class instances (Server Actions are the exception, section 20).
3. **A Client Component cannot import a Server Component**, but it **can render one passed as `children`** or another prop:

```tsx
// app/dashboard/layout.tsx (server)
<SidebarToggle>          {/* client: holds open/closed state */}
  <SidebarNav />         {/* server: still rendered on the server */}
</SidebarToggle>
```

4. **Context providers are Client Components.** Wrap them in one `providers.tsx` file and render it in the root layout around `{children}`; the pages inside stay Server Components.
5. **Protect server-only code:** `import 'server-only'` at the top of `lib/db.ts` makes the build fail if a Client Component ever imports it. (`'client-only'` does the opposite.)

## 18.3 Rendering strategies

| Strategy | When the HTML is made | PharmaCare page |
| --- | --- | --- |
| **Static (SSG)** | At build time, served from cache/CDN | About, branches, medicine detail pages |
| **ISR** (incremental static regeneration) | Static, then refreshed in the background after a time or on demand | Medicine catalogue updated every hour |
| **Dynamic (SSR)** | On every request | Staff dashboard, anything using cookies or the logged-in user |
| **Client-side (CSR)** | In the browser after JS loads | Live POS cart, charts |
| **Streaming** | Server sends the page in chunks as parts become ready | Dashboard: shell first, slow reports stream in |
| **Partial prerendering (PPR)** | A static shell from the cache + dynamic holes streamed in, in one response | Product page: static details + live stock |

**Static vs dynamic:** a route becomes dynamic when it reads request-time data: `cookies()`, `headers()`, `searchParams`, `connection()`, or uncached data. Otherwise Next.js can render it at build time.

## 18.4 Cache Components and partial prerendering (Next.js 16)

**What it is:** With `cacheComponents: true`, Next.js 16 uses one clear model:

- Everything is **dynamic by default** (fresh on each request).
- You **opt in** to caching with the `'use cache'` directive (section 19).
- Each route gets a **static shell** prerendered at build time; dynamic parts wrapped in `<Suspense>` are **streamed** into that shell at request time. This is partial prerendering.

```ts
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  cacheComponents: true,
  reactCompiler: true,
};
export default nextConfig;
```

```tsx
// app/medicines/[slug]/page.tsx
import { Suspense } from 'react';

export default async function MedicinePage({ params }: PageProps<'/medicines/[slug]'>) {
  const { slug } = await params;
  return (
    <article>
      <MedicineInfo slug={slug} />                         {/* cached with 'use cache' → in the static shell */}
      <Suspense fallback={<p className="text-gray-400">Checking stock…</p>}>
        <LiveStock slug={slug} />                          {/* dynamic → streamed per request */}
      </Suspense>
    </article>
  );
}
```

If a component reads dynamic data without being cached or wrapped in `<Suspense>`, Next.js tells you during development, so the static shell is never blocked by accident.

## 18.5 Streaming with Suspense

**What it is:** Instead of waiting for the slowest query before sending anything, Next.js sends the page in pieces. Each `<Suspense>` boundary shows its fallback immediately and fills in when its data is ready. `loading.tsx` is a Suspense boundary around the whole page.

```tsx
// app/dashboard/page.tsx
export default function Dashboard() {
  return (
    <div className="grid gap-6 lg:grid-cols-3">
      <Suspense fallback={<CardSkeleton />}><TodaySales /></Suspense>       {/* 50 ms */}
      <Suspense fallback={<CardSkeleton />}><LowStock /></Suspense>         {/* 200 ms */}
      <Suspense fallback={<CardSkeleton />}><MonthlyReport /></Suspense>    {/* 2 s — no longer blocks the rest */}
    </div>
  );
}

async function TodaySales() {
  const total = await getTodaySalesTotal();
  return <StatCard label="Sales today" value={`${total.toLocaleString()} TZS`} />;
}
```

## 18.6 Runtimes: Node.js and Edge

| Runtime | What it is | Use |
| --- | --- | --- |
| **Node.js** (default) | Full Node.js APIs and npm packages | Almost everything: database drivers, PDF generation |
| **Edge** | Lightweight runtime close to users, limited APIs | Very simple, latency-sensitive logic. Not supported with Cache Components |

In Next.js 16, `proxy.ts` runs on Node.js. Stay on Node.js unless you have a specific reason.

## 18.7 Choosing the right strategy: decision guide

1. Same for every visitor and changes rarely → **static** (`'use cache'` + `cacheLife('max')` or default static).
2. Same for everyone but changes sometimes → **cached with a lifetime or tag** (`cacheLife('hours')`, revalidate on change).
3. Depends on the user or request (cookies, session) → **dynamic**, inside `<Suspense>`.
4. Changes as the user interacts → **Client Component** state.
5. Slow piece on a fast page → wrap it in **`<Suspense>`** and stream it.
