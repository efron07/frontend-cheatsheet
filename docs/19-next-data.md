# 19. Data fetching and caching

## 19.1 Where to fetch data

| Location | How | Use for |
| --- | --- | --- |
| **Server Component** | `await` in the component | Default: pages, lists, detail views |
| **Server Action** | `'use server'` function | Writes (create sale, receive stock) — section 20 |
| **Route Handler** | `app/api/.../route.ts` | Webhooks, mobile app, third parties — section 20 |
| **Client Component** | TanStack Query / SWR, or `use()` on a promise | Live, frequently-changing, interactive data (POS search) |

## 19.2 Fetching in Server Components

```ts
// lib/data/medicines.ts
import 'server-only';

const API = process.env.API_URL!;           // server-only variable: never sent to the browser

export async function getMedicines(search = ''): Promise<Medicine[]> {
  const res = await fetch(`${API}/medicines?search=${encodeURIComponent(search)}`, {
    headers: { Authorization: `Bearer ${process.env.API_TOKEN}`, Accept: 'application/json' },
  });
  if (!res.ok) throw new Error(`Failed to load medicines (${res.status})`);
  const json = await res.json();
  return MedicineSchema.array().parse(json.data);    // validate the shape (section 9.9)
}
```

```tsx
// app/medicines/page.tsx
export default async function Page({ searchParams }: PageProps<'/medicines'>) {
  const { search = '' } = await searchParams;
  const medicines = await getMedicines(String(search));
  return <MedicineGrid medicines={medicines} />;
}
```

You can also query a database directly with Prisma or Drizzle (section 22). No API layer is required when Next.js owns the data.

## 19.3 Parallel vs sequential fetching

```tsx
// Sequential (waterfall): second waits for the first — slow
const medicine = await getMedicine(slug);
const batches = await getBatches(medicine.id);     // OK only because it needs medicine.id

// Parallel: independent requests start together
const [categories, branches, featured] = await Promise.all([getCategories(), getBranches(), getFeatured()]);
```

Even better: split independent data into separate components inside `<Suspense>` so each streams on its own (section 18.5).

## 19.4 Preloading data

Start a fetch early, before the component that needs it renders:

```ts
export const preloadMedicine = (slug: string) => { void getMedicine(slug); };

// page.tsx
preloadMedicine(slug);                // kick off now
const user = await getUser();         // meanwhile do something else
return <MedicineDetail slug={slug} />; // getMedicine(slug) inside reuses the in-flight request
```

## 19.5 Request memoisation with React `cache`

**What it is:** During **one** render, several components may need the same data (the layout needs the user, the page needs the user). Wrap the function in React's `cache()` and it runs only once per request. `fetch` GET requests are de-duplicated this way automatically.

```ts
import { cache } from 'react';

export const getCurrentUser = cache(async () => {
  const session = await getSession();
  return session ? db.user.findUnique({ where: { id: session.userId } }) : null;
});
```

This is per request only. For data shared **across** requests and users, use `'use cache'`.

## 19.6 Caching with `'use cache'` (Next.js 16)

**What it is:** Mark a function, component or whole page with `'use cache'` and Next.js stores its result and reuses it for later requests. With Cache Components, nothing is cached unless you say so.

```ts
// lib/data/medicines.ts
import { cacheLife, cacheTag } from 'next/cache';

export async function getCatalogue() {
  'use cache';
  cacheLife('hours');                    // how long it stays fresh
  cacheTag('medicines');                 // a label so we can clear it later
  return db.medicine.findMany({ include: { category: true }, orderBy: { name: 'asc' } });
}

export async function getMedicine(slug: string) {
  'use cache';
  cacheLife('days');
  cacheTag('medicines', `medicine-${slug}`);   // arguments become part of the cache key
  return db.medicine.findUnique({ where: { slug }, include: { category: true } });
}
```

```tsx
// Cache a whole component
async function BranchList() {
  'use cache';
  cacheLife('max');
  const branches = await db.branch.findMany();
  return <ul>{branches.map((b) => <li key={b.id}>{b.name} — {b.phone}</li>)}</ul>;
}
```

**Built-in `cacheLife` profiles:** `'seconds'`, `'minutes'`, `'hours'`, `'days'`, `'weeks'`, `'max'`, or a custom object `cacheLife({ stale: 60, revalidate: 300, expire: 3600 })` (seconds). The default without `cacheLife` is short (minutes).

**Rules:**

- You **cannot** read `cookies()`, `headers()` or `searchParams` inside `'use cache'`. Read them outside and pass the values in as arguments.
- Arguments and return values must be serialisable.
- `'use cache: private'` allows per-user caching; `'use cache: remote'` uses a shared platform cache (e.g. Redis).

## 19.7 Revalidating (clearing) cached data

When data changes — a pharmacist edits a price — you must clear the cached copy.

| Function | Effect | Use in |
| --- | --- | --- |
| `updateTag('medicines')` | Expire now; the **same** request sees fresh data (read-your-writes) | Server Actions after a staff edit |
| `revalidateTag('medicines', 'max')` | Mark stale; serve old data while refreshing in the background | Webhooks, Route Handlers, less urgent updates |
| `revalidatePath('/medicines')` | Clear everything for a path | When you do not use tags |
| `refresh()` | Refresh the client's current page data from a Server Action | After non-cached updates |

```ts
// app/dashboard/medicines/actions.ts
'use server';
import { updateTag } from 'next/cache';

export async function updatePrice(id: number, slug: string, price: number) {
  await db.medicine.update({ where: { id }, data: { price } });
  updateTag(`medicine-${slug}`);
  updateTag('medicines');
}
```

```ts
// app/api/webhooks/stock/route.ts — Laravel tells Next.js stock changed
import { revalidateTag } from 'next/cache';

export async function POST(req: Request) {
  if (req.headers.get('x-webhook-secret') !== process.env.WEBHOOK_SECRET) return new Response('Forbidden', { status: 403 });
  const { slug } = await req.json();
  revalidateTag(`medicine-${slug}`, 'max');
  return Response.json({ ok: true });
}
```

**Errors during revalidation:** if a background refresh throws, Next.js keeps serving the last good cached version and retries later, so a failing API does not break the page.

## 19.8 Caching without Cache Components

Many existing projects and tutorials use the earlier model. You will recognise it:

```ts
await fetch(url, { cache: 'force-cache' });               // cache forever
await fetch(url, { next: { revalidate: 3600, tags: ['medicines'] } });   // ISR: refresh hourly
await fetch(url, { cache: 'no-store' });                  // always fresh

export const revalidate = 3600;          // route segment config: whole page
export const dynamic = 'force-dynamic';  // always render per request
```

With `cacheComponents: true`, these map to `'use cache'` + `cacheLife(...)`; `unstable_cache()` is replaced by `'use cache'`.

## 19.9 Client-side data and the `use()` hook

Pass a promise from a Server Component to a Client Component without awaiting it; the client unwraps it with `use()` inside Suspense. The page starts streaming immediately.

```tsx
// page.tsx (server)
export default function Page() {
  const salesPromise = getTodaySales();               // not awaited
  return (
    <Suspense fallback={<TableSkeleton />}>
      <LiveSalesTable salesPromise={salesPromise} />
    </Suspense>
  );
}

// live-sales-table.tsx (client)
'use client';
import { use } from 'react';
export function LiveSalesTable({ salesPromise }: { salesPromise: Promise<Sale[]> }) {
  const sales = use(salesPromise);
  const [filter, setFilter] = useState<PaymentMethod | 'all'>('all');
  /* interactive filtering on the client */
}
```

For data that changes while the user watches (POS search, payment status), use TanStack Query in Client Components (section 15.3).
