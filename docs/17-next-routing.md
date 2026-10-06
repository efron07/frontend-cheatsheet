# 17. Next.js fundamentals and routing

## 17.1 What Next.js is and why use it

**What it is:** Next.js is a **React framework**. React gives you components; Next.js adds everything a real website needs: file-based routing, server rendering, data fetching, caching, image and font optimisation, API endpoints, SEO metadata and production builds.

**SPA vs server rendering:**

|  | SPA / client-side rendering (CSR) | Server-side rendering (SSR) / static (SSG) |
| --- | --- | --- |
| First load | Empty HTML + big JS bundle, then render | Full HTML arrives ready to read |
| SEO | Weak (crawlers see little) | Strong |
| Speed on slow phones | Slow (all work on the device) | Fast first paint |
| Example | Vite + React | Next.js |

**Why Next.js for PharmaCare:** the public medicine catalogue must be found on Google (server-rendered); the staff POS must be highly interactive (client components). Next.js does both in one app. Alternatives: React Router (framework mode, formerly Remix), TanStack Start, Astro.

**App Router vs Pages Router:** Next.js has two routers. The **App Router** (`app/` folder) is the modern one, built on React Server Components — use it for all new work. The **Pages Router** (`pages/` folder, `getServerSideProps`) is the older one you will see in existing projects.

## 17.2 Create a project

```bash
npx create-next-app@latest pharmacare-web
# Answer: TypeScript Yes, ESLint (or Biome) Yes, Tailwind Yes, src/ Yes, App Router Yes,
#         Turbopack Yes, import alias @/* Yes
cd pharmacare-web
npm run dev            # http://localhost:3000 (Turbopack is the default bundler in Next.js 16)
npm run build          # production build
npm start              # run the production build
```

Next.js 16 requires Node.js 20.9 or newer.

## 17.3 Folder structure

| Path | What it is |
| --- | --- |
| `src/app/` | Routes: every folder is a URL segment |
| `src/app/layout.tsx` | Root layout: `<html>` and `<body>`, wraps every page |
| `src/app/page.tsx` | The home page `/` |
| `src/app/globals.css` | Global styles + `@import 'tailwindcss'` |
| `src/components/` | Shared React components (`ui/` for shadcn/ui) |
| `src/lib/` | Utilities, API client, database client |
| `src/hooks/`, `src/stores/`, `src/types/` | Custom hooks, Zustand stores, TS types |
| `public/` | Static files served at `/` (favicon, `robots.txt`, images) |
| `next.config.ts` | Next.js configuration |
| `proxy.ts` | Runs before requests (auth redirects) — called `middleware.ts` before Next.js 16 |
| `.env.local` | Local secrets (never committed) |

## 17.4 Routing basics

**What it is:** Folders define URL segments; a `page.tsx` file makes that segment publicly visible.

```
src/app/
├── layout.tsx                    → wraps everything
├── page.tsx                      → /
├── medicines/
│   ├── page.tsx                  → /medicines
│   └── [slug]/
│       └── page.tsx              → /medicines/amoxicillin-500mg
├── branches/page.tsx             → /branches
└── dashboard/
    ├── layout.tsx                → sidebar for all /dashboard pages
    ├── page.tsx                  → /dashboard
    ├── pos/page.tsx              → /dashboard/pos
    └── stock/page.tsx            → /dashboard/stock
```

**Routing terminology:** a **segment** is one part of the URL path (`medicines`); a **route** is the full path; **dynamic segments** are filled from the URL; a **layout** wraps child routes; the **root** is `app/`.

**Special files in each route folder:**

| File | Purpose |
| --- | --- |
| `page.tsx` | The UI for this URL (makes the route public) |
| `layout.tsx` | Shared UI that wraps this segment and its children; keeps state between navigations |
| `template.tsx` | Like a layout, but remounts on every navigation |
| `loading.tsx` | Loading UI shown instantly while the page loads (wraps the page in Suspense) |
| `error.tsx` | Error UI if something in this segment throws (must be a Client Component) |
| `not-found.tsx` | UI for `notFound()` or unknown URLs |
| `global-error.tsx` | Error UI for the root layout itself |
| `route.ts` | An API endpoint instead of a page (section 20) |
| `default.tsx` | Fallback for parallel routes |

## 17.5 Pages and layouts

```tsx
// src/app/layout.tsx — the root layout (required)
import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import './globals.css';

const inter = Inter({ subsets: ['latin'], variable: '--font-sans' });

export const metadata: Metadata = {
  title: { default: 'PharmaCare', template: '%s | PharmaCare' },
  description: 'Genuine medicines in Dodoma. Check availability online.',
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={inter.variable}>
      <body className="min-h-dvh bg-gray-50 font-sans text-gray-900 antialiased">{children}</body>
    </html>
  );
}
```

```tsx
// src/app/dashboard/layout.tsx — nested layout
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="grid min-h-dvh lg:grid-cols-[240px_1fr]">
      <Sidebar />
      <main className="p-6">{children}</main>
    </div>
  );
}
```

```tsx
// src/app/medicines/page.tsx — a page (a Server Component by default)
export const metadata = { title: 'Medicines' };

export default async function MedicinesPage() {
  const medicines = await getMedicines();        // runs on the server
  return (
    <section className="mx-auto max-w-6xl p-6">
      <h1 className="mb-6 text-3xl font-bold">Medicines</h1>
      <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-4">
        {medicines.map((m) => <MedicineCard key={m.id} medicine={m} />)}
      </div>
    </section>
  );
}
```

## 17.6 Dynamic routes

```tsx
// src/app/medicines/[slug]/page.tsx
import { notFound } from 'next/navigation';

export default async function MedicinePage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;                 // params is a Promise in Next.js 15+
  const medicine = await getMedicineBySlug(slug);
  if (!medicine) notFound();                     // shows not-found.tsx with a 404

  return <MedicineDetail medicine={medicine} />;
}

// Pre-build these pages at build time (static generation)
export async function generateStaticParams() {
  const medicines = await getAllMedicines();
  return medicines.map((m) => ({ slug: m.slug }));
}
```

Next.js also generates the `PageProps<'/medicines/[slug]'>` helper type, so you can write `props: PageProps<'/medicines/[slug]'>`.

| Pattern | Folder | Matches |
| --- | --- | --- |
| Dynamic | `[slug]` | `/medicines/amoxicillin` |
| Catch-all | `[...path]` | `/docs/a`, `/docs/a/b/c` |
| Optional catch-all | `[[...path]]` | `/docs` too |

**Search params** (`/medicines?search=amox&page=2`):

```tsx
export default async function Page({ searchParams }: { searchParams: Promise<{ search?: string; page?: string }> }) {
  const { search = '', page = '1' } = await searchParams;
  const results = await searchMedicines(search, Number(page));
  /* ... */
}
```

## 17.7 Organising routes

- **Route groups** `(name)` — organise folders or give sections different layouts **without** changing the URL: `app/(public)/medicines` and `app/(staff)/dashboard`, each group with its own `layout.tsx`.
- **Private folders** `_name` — not routable; colocate components: `app/dashboard/_components/sales-chart.tsx`.
- **Colocation** — any file that is not a special file (`page`, `route`…) is not public, so you can keep components next to the route that uses them.

```
app/
├── (public)/
│   ├── layout.tsx          → store header and footer
│   ├── page.tsx            → /
│   └── medicines/...       → /medicines
└── (staff)/
    └── dashboard/
        ├── layout.tsx      → sidebar, requires login
        └── _components/    → private, not a route
```

## 17.8 Advanced routing: parallel and intercepting routes

**Parallel routes** — render several independent pages in one layout using **slots** named `@folder`. Each slot loads and errors independently.

```
app/dashboard/
├── layout.tsx           → receives { children, sales, stock }
├── page.tsx
├── @sales/page.tsx      → today's sales widget
└── @stock/page.tsx      → low-stock widget
```

```tsx
export default function Layout({ children, sales, stock }: { children: React.ReactNode; sales: React.ReactNode; stock: React.ReactNode }) {
  return <div className="grid gap-6 lg:grid-cols-2">{sales}{stock}{children}</div>;
}
```

**Intercepting routes** — show a route inside the current page (usually a modal) when navigating in-app, but as a full page when opened directly or refreshed. Folder prefixes: `(.)` same level, `(..)` one level up, `(...)` from the root. Example: clicking a medicine in the grid opens `app/medicines/(.)[slug]/page.tsx` as a quick-view modal, while sharing the URL opens the full page.

## 17.9 Navigation

```tsx
import Link from 'next/link';

// <Link> navigates without a full reload and prefetches the page when visible
<Link href="/medicines">Medicines</Link>
<Link href={`/medicines/${m.slug}`}>{m.name}</Link>
<Link href={{ pathname: '/medicines', query: { category: 'antibiotics' } }}>Antibiotics</Link>
```

```tsx
'use client';
import { useRouter, usePathname, useSearchParams } from 'next/navigation';

export function CategoryFilter() {
  const router = useRouter();
  const pathname = usePathname();          // "/medicines"
  const searchParams = useSearchParams();  // read ?category=...

  function setCategory(category: string) {
    const params = new URLSearchParams(searchParams);
    category ? params.set('category', category) : params.delete('category');
    params.delete('page');
    router.push(`${pathname}?${params.toString()}`);   // also: router.replace, router.back, router.refresh
  }
  /* ... */
}
```

```tsx
// Server-side redirect (in Server Components, Server Actions, route handlers)
import { redirect, permanentRedirect } from 'next/navigation';
redirect('/login');

// Highlight the active link
const isActive = pathname.startsWith('/dashboard/stock');
```

**Redirects and rewrites in config** (old URLs, vanity URLs):

```ts
// next.config.ts
async redirects() {
  return [{ source: '/products/:slug', destination: '/medicines/:slug', permanent: true }];
},
```

## 17.10 Loading and error states

```tsx
// app/medicines/loading.tsx — shown instantly while page.tsx loads
export default function Loading() {
  return (
    <div className="grid gap-4 p-6 sm:grid-cols-2 lg:grid-cols-4">
      {Array.from({ length: 8 }, (_, i) => <div key={i} className="h-64 animate-pulse rounded-xl bg-gray-200" />)}
    </div>
  );
}
```

```tsx
// app/medicines/error.tsx — must be a Client Component
'use client';
export default function Error({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  return (
    <div role="alert" className="p-6">
      <h2 className="font-semibold">Could not load medicines</h2>
      <button onClick={reset} className="mt-3 underline">Try again</button>
    </div>
  );
}
```

```tsx
// app/not-found.tsx
import Link from 'next/link';
export default function NotFound() {
  return <div className="p-6">Page not found. <Link href="/medicines">Browse medicines</Link></div>;
}
```
