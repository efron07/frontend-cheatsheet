# 27. Quick reference

## 27.1 Commands

| Command | Does | Section |
| --- | --- | --- |
| `npx create-next-app@latest app-name` | New Next.js project | 17.2 |
| `npm create vite@latest app -- --template react-ts` | New Vite React + TS app | 12.2 |
| `npm run dev` / `build` / `start` | Develop / build / run production | 17.2 |
| `npx tsc --noEmit` | Type-check only | 7.2 |
| `npx tsx script.ts` | Run a TS file | 7.2 |
| `npx eslint .` / `npx prettier --write .` | Lint / format | 25.4 |
| `npx shadcn@latest init` / `add button` | Set up / add shadcn/ui components | 16.2 |
| `npx vitest` / `npx playwright test` | Unit / E2E tests | 26.2–26.3 |
| `npx playwright codegen localhost:3000` | Record an E2E test | 26.3 |
| `npx drizzle-kit generate` / `migrate` | DB migrations | 22.1 |
| `npx @next/codemod@latest upgrade` | Upgrade Next.js with codemods | 25.3 |
| `npm audit` | Check dependency vulnerabilities | 25.3 |
| `git switch -c feature/x` | New branch | 25.2 |

## 27.2 JavaScript one-liners

```js
const total = cart.reduce((s, i) => s + i.price * i.qty, 0);
const inStock = medicines.filter((m) => m.stock > 0);
const byId = new Map(medicines.map((m) => [m.id, m]));
const grouped = Object.groupBy(medicines, (m) => m.category);
const unique = [...new Set(list)];
const sorted = medicines.toSorted((a, b) => a.price - b.price);
const copy = structuredClone(sale);
const fmt = new Intl.NumberFormat('en-TZ').format(1500);     // "1,500"
const [a, b] = await Promise.all([getA(), getB()]);
const name = sale.customer?.name ?? 'Walk-in';
```

## 27.3 TypeScript patterns

```ts
type PaymentMethod = 'cash' | 'mpesa' | 'airtel' | 'card';
type NewMedicine = Omit<Medicine, 'id'>;
type MedicineUpdate = Partial<NewMedicine>;
type Totals = Record<PaymentMethod, number>;
type Result<T> = { ok: true; data: T } | { ok: false; error: string };
const LABELS = { cash: 'Cash' } satisfies Partial<Record<PaymentMethod, string>>;
type Medicine = z.infer<typeof MedicineSchema>;
```

## 27.4 React patterns

```tsx
const [items, setItems] = useState<CartItem[]>([]);
setItems((prev) => [...prev, item]);                                        // add
setItems((prev) => prev.map((i) => (i.id === id ? { ...i, qty } : i)));     // update
setItems((prev) => prev.filter((i) => i.id !== id));                        // remove

useEffect(() => {
  const id = setInterval(poll, 3000);
  return () => clearInterval(id);                                           // always clean up
}, [poll]);

{list.length > 0 && <List items={list} />}
{items.map((i) => <Row key={i.id} item={i} />)}
```

## 27.5 Next.js file conventions

| File | Purpose |
| --- | --- |
| `page.tsx` | Route UI |
| `layout.tsx` | Shared wrapper |
| `loading.tsx` | Instant loading UI |
| `error.tsx` | Error UI (client) |
| `not-found.tsx` | 404 UI |
| `route.ts` | API endpoint |
| `proxy.ts` | Runs before requests |
| `sitemap.ts`, `robots.ts`, `manifest.ts` | SEO and PWA files |
| `opengraph-image.tsx`, `icon.png` | Share image, favicon |
| `instrumentation.ts` | Monitoring setup |
| `[slug]`, `[...slug]`, `(group)`, `_private`, `@slot`, `(.)intercept` | Folder conventions |

## 27.6 Next.js snippets

```tsx
// Server Component page with params
export default async function Page({ params }: PageProps<'/medicines/[slug]'>) {
  const { slug } = await params;
  const m = await getMedicine(slug);
  if (!m) notFound();
  return <MedicineDetail medicine={m} />;
}

// Cached data function
export async function getMedicine(slug: string) {
  'use cache';
  cacheLife('days');
  cacheTag(`medicine-${slug}`);
  return db.medicine.findUnique({ where: { slug } });
}

// Server Action
'use server';
export async function updatePrice(id: number, slug: string, formData: FormData) {
  await requireRole(['admin', 'pharmacist']);
  const { price } = z.object({ price: z.coerce.number().int().min(0) }).parse(Object.fromEntries(formData));
  await db.medicine.update({ where: { id }, data: { price } });
  updateTag(`medicine-${slug}`);
}

// Route Handler
export async function GET(req: NextRequest) {
  const q = req.nextUrl.searchParams.get('q') ?? '';
  return Response.json({ data: await search(q) });
}

// Metadata
export async function generateMetadata({ params }: PageProps<'/medicines/[slug]'>): Promise<Metadata> {
  const m = await getMedicine((await params).slug);
  return { title: m?.name ?? 'Not found' };
}
```

## 27.7 Tailwind cheats

| Need | Classes |
| --- | --- |
| Centre anything | `grid place-items-center` or `flex items-center justify-center` |
| Responsive grid | `grid gap-4 sm:grid-cols-2 lg:grid-cols-4` |
| Container | `mx-auto max-w-6xl px-4` |
| Sidebar layout | `grid lg:grid-cols-[240px_1fr]` |
| Truncate text | `truncate` or `line-clamp-2` |
| Card | `rounded-xl border bg-white p-4 shadow-sm` |
| Primary button | `rounded-md bg-brand-600 px-4 py-2 text-white hover:bg-brand-700 disabled:opacity-50` |
| Visible focus | `focus-visible:outline-2 focus-visible:outline-offset-2` |
| Skeleton | `animate-pulse rounded bg-gray-200` |
| Dark mode | `bg-white dark:bg-gray-900` |
| Screen-reader only text | `sr-only` |

## 27.8 Decision guide

| Question | Answer |
| --- | --- |
| Where do I fetch data? | Server Component; client only for live/interactive data (TanStack Query) |
| How do I save data? | Server Action (own UI) or Route Handler (external callers) |
| Where does state live? | Component → parent → context/Zustand → URL → server |
| Do I need `'use client'`? | Only for state, effects, events or browser APIs — on the smallest component |
| Do I need `useEffect`? | Only to sync with something outside React |
| Do I need `useMemo`? | Usually not with the React Compiler; measure first |
| How do I validate? | One Zod schema, used in the form and on the server |
| How do I cache? | `'use cache'` + `cacheLife` + `cacheTag`; `updateTag` after writes |
