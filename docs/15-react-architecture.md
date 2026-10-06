# 15. React app architecture

## 15.1 Composition patterns

**Composition** means building complex UI by putting simple components together, instead of making one component do everything with dozens of props.

**Pass components instead of drilling props:**

```tsx
// Instead of <Layout user={user} cartCount={n} onLogout={...} /> passing through 4 levels:
<DashboardLayout sidebar={<Sidebar user={user} />} header={<Header cartCount={n} />}>
  <SalesTable />
</DashboardLayout>
```

**Compound components** — a group of components that work together and share state through context:

```tsx
<Tabs defaultValue="stock">
  <Tabs.List>
    <Tabs.Trigger value="stock">Stock</Tabs.Trigger>
    <Tabs.Trigger value="batches">Batches</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Panel value="stock"><StockTable /></Tabs.Panel>
  <Tabs.Panel value="batches"><BatchTable /></Tabs.Panel>
</Tabs>
```

**Older patterns you will meet in existing code:**

- **Higher-Order Component (HOC):** a function that takes a component and returns an enhanced one: `export default withAuth(Dashboard)`. Mostly replaced by hooks.
- **Render props:** a prop that is a function returning JSX: `<MouseTracker render={(pos) => <Tooltip at={pos} />} />`. Also mostly replaced by hooks.

**Headless components:** components that provide behaviour and accessibility (keyboard, focus, ARIA) but **no styling**, so you style them with Tailwind. Examples: Radix UI, React Aria, Ark UI, Headless UI.

## 15.2 State management libraries

You do not need a library for most apps: `useState` + context + a server cache covers a lot. Reach for one when global client state becomes complex or changes often.

| Library | Style | When |
| --- | --- | --- |
| **Zustand** | Small hook-based store | Default pick: POS cart, UI state shared across pages |
| **Jotai** | Atoms (tiny pieces of state) | Many small independent values |
| **Redux Toolkit** (+ RTK Query) | Central store, actions, reducers, devtools | Large teams, complex rules, existing Redux code |
| **MobX** | Observable objects, auto-tracking | Teams that like mutable OOP style |
| **XState** | State machines | Complex flows (multi-step payment) |

**Zustand example — the POS cart:**

```bash
npm install zustand
```

```ts
// stores/cart-store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface CartState {
  items: CartItem[];
  add: (m: Medicine) => void;
  setQty: (id: number, qty: number) => void;
  remove: (id: number) => void;
  clear: () => void;
}

export const useCartStore = create<CartState>()(
  persist(
    (set) => ({
      items: [],
      add: (m) => set((s) => ({ items: addLine(s.items, m) })),
      setQty: (id, qty) => set((s) => ({ items: s.items.map((i) => (i.medicine.id === id ? { ...i, qty } : i)) })),
      remove: (id) => set((s) => ({ items: s.items.filter((i) => i.medicine.id !== id) })),
      clear: () => set({ items: [] }),
    }),
    { name: 'pharmacare-cart' },        // saved to localStorage automatically
  ),
);

// Selectors: a component re-renders only when ITS slice changes
export const useCartTotal = () =>
  useCartStore((s) => s.items.reduce((t, i) => t + i.medicine.price * i.qty, 0));
```

```tsx
function AddButton({ medicine }: { medicine: Medicine }) {
  const add = useCartStore((s) => s.add);
  return <button onClick={() => add(medicine)}>Add</button>;
}

function CartTotal() {
  const total = useCartTotal();
  return <strong>{total.toLocaleString()} TZS</strong>;
}
```

## 15.3 Data fetching with TanStack Query

**What it is:** Server data (medicines, sales) is not like UI state: it lives on the server, can go stale, and many components need it. **TanStack Query** (React Query) is a server-state cache: it fetches, caches, de-duplicates, refetches in the background, retries and handles loading/error states. **SWR** is a lighter alternative; **RTK Query** comes with Redux; **Apollo**, **Relay** and **urql** do the same for GraphQL APIs.

In Next.js, prefer fetching in **Server Components** (section 19). Use TanStack Query in client-heavy parts (the POS) or plain React apps.

```bash
npm install @tanstack/react-query
```

```tsx
// app/providers.tsx
'use client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  const [client] = useState(() => new QueryClient({ defaultOptions: { queries: { staleTime: 60_000 } } }));
  return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
}
```

```tsx
// hooks/use-medicines.ts
import { useQuery, useMutation, useQueryClient, keepPreviousData } from '@tanstack/react-query';

export function useMedicines(search: string, page = 1) {
  return useQuery({
    queryKey: ['medicines', { search, page }],          // cache key: changes → refetch
    queryFn: () => api<Paginated<Medicine>>(`/medicines?search=${search}&page=${page}`),
    placeholderData: keepPreviousData,                   // keep old page while the next loads
    enabled: search.length === 0 || search.length >= 2,
  });
}

export function useCreateSale() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (input: CreateSaleInput) => api<Sale>('/sales', { method: 'POST', body: input }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['medicines'] });   // stock changed: refetch
      queryClient.invalidateQueries({ queryKey: ['sales', 'today'] });
    },
  });
}
```

```tsx
function MedicineSearchResults({ search }: { search: string }) {
  const { data, isPending, isError, error, isFetching } = useMedicines(search);

  if (isPending) return <SkeletonList />;
  if (isError) return <p className="text-red-600">{error.message}</p>;

  return (
    <div className={isFetching ? 'opacity-60' : ''}>
      {data.data.map((m) => <MedicineRow key={m.id} medicine={m} onAdd={add} />)}
    </div>
  );
}

function CheckoutButton() {
  const { mutate, isPending } = useCreateSale();
  const items = useCartStore((s) => s.items);
  return (
    <button disabled={isPending || items.length === 0}
      onClick={() => mutate({ items: items.map(toLine), paymentMethod: 'cash' }, { onSuccess: () => useCartStore.getState().clear() })}>
      {isPending ? 'Saving…' : 'Complete sale'}
    </button>
  );
}
```

## 15.4 Routing in plain React apps

**What it is:** Showing different components for different URLs without a full page reload. Next.js has routing built in (section 17). For Vite React apps, use **React Router** or **TanStack Router** (fully type-safe).

```tsx
// main.tsx — React Router
import { createBrowserRouter, RouterProvider, Link, useParams, Outlet } from 'react-router';

const router = createBrowserRouter([
  {
    path: '/',
    element: <AppLayout />,              // renders <Outlet /> for children
    children: [
      { index: true, element: <Dashboard /> },
      { path: 'medicines', element: <MedicineList /> },
      { path: 'medicines/:id', element: <MedicineDetail /> },
      { path: '*', element: <NotFound /> },
    ],
  },
]);

function MedicineDetail() {
  const { id } = useParams();
  /* load medicine by id */
}

<RouterProvider router={router} />;
```

## 15.5 Suspense

**What it is:** `<Suspense>` shows a fallback (spinner, skeleton) while something inside it is **not ready yet**: lazy-loaded code or data being fetched. It lets you declare loading states in the layout instead of `if (loading)` in every component.

```tsx
import { Suspense, lazy } from 'react';

const SalesChart = lazy(() => import('./sales-chart'));     // code-split: loaded only when shown

<Suspense fallback={<div className="h-64 animate-pulse rounded bg-gray-100" />}>
  <SalesChart />
</Suspense>
```

In Next.js, Suspense powers **streaming**: slow parts of a page load in while the rest is already visible (section 18).

## 15.6 Error boundaries

**What it is:** A component that catches rendering errors in its children and shows a fallback instead of a blank white screen. It must be a class component, so use the `react-error-boundary` package (Next.js gives you `error.tsx` files instead, section 17).

```tsx
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary fallbackRender={({ error, resetErrorBoundary }) => (
  <div role="alert">
    <p>Sales chart failed: {error.message}</p>
    <button onClick={resetErrorBoundary}>Try again</button>
  </div>
)}>
  <SalesChart />
</ErrorBoundary>
```

Error boundaries do not catch errors in event handlers or async code; handle those with `try/catch`.

## 15.7 Portals

**What it is:** Render a component into a different place in the DOM (usually `document.body`) while it stays a child in React. Used for modals, toasts and dropdowns so parent `overflow: hidden` or `z-index` cannot clip them.

```tsx
import { createPortal } from 'react-dom';

function Modal({ open, children }: { open: boolean; children: React.ReactNode }) {
  if (!open) return null;
  return createPortal(
    <div className="fixed inset-0 z-50 grid place-items-center bg-black/40">
      <div role="dialog" aria-modal="true" className="rounded-xl bg-white p-6">{children}</div>
    </div>,
    document.body,
  );
}
```

In practice use a headless library's `Dialog` (Radix, shadcn/ui): it handles focus trapping, Escape to close and screen readers.

## 15.8 Calling APIs: `fetch`, Axios, server APIs

- **`fetch`** — built in; wrap it once in an API client (section 5.8).
- **Axios** — interceptors (attach token, refresh on 401), automatic JSON, throws on errors.
- **GraphQL clients** — Apollo, urql, Relay.
- **Server APIs** — in Next.js, Server Components and Server Actions talk to your database or Laravel API directly, so many client-side fetches disappear.

## 15.9 Beyond the web

- **React Native / Expo** — build iOS and Android apps with the same React skills (a PharmaCare stock-taking app).
- **Astro** — content sites that ship almost no JS, with React "islands" where needed.
- **Electron / Tauri** — desktop apps (an offline POS).
