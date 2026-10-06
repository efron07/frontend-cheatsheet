# 14. Hooks

**What it is:** Hooks are functions starting with `use` that let function components use React features: state, side effects, context, refs and more.

## 14.1 Rules of hooks

1. **Only call hooks at the top level** of a component or custom hook — never inside `if`, loops, nested functions or after an early `return`. React identifies each hook by its call order.
2. **Only call hooks from React functions** (components or custom hooks), not from plain functions.

The ESLint plugin `eslint-plugin-react-hooks` enforces both. Keep it on.

## 14.2 Component lifecycle

A component **mounts** (appears), **updates** (re-renders when props/state/context change), and **unmounts** (disappears). In function components you do not use lifecycle methods; you synchronise with the outside world using effects, which run after mount/updates and clean up before the next run and on unmount.

## 14.3 `useEffect` — syncing with the outside world

**What it is:** Runs code **after** React has updated the screen, to synchronise with something outside React: a subscription, a timer, a browser API, a non-React widget.

```tsx
import { useEffect, useState } from 'react';

// Keyboard shortcut for the POS: F2 focuses search
function usePosShortcuts(onFocusSearch: () => void) {
  useEffect(() => {
    function onKey(e: KeyboardEvent) {
      if (e.key === 'F2') { e.preventDefault(); onFocusSearch(); }
    }
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);   // CLEANUP: runs on unmount/re-run
  }, [onFocusSearch]);                                            // dependency array
}

// Poll M-Pesa payment status every 3 seconds while pending
function usePaymentStatus(saleId: number | null) {
  const [status, setStatus] = useState<'pending' | 'paid' | 'failed'>('pending');

  useEffect(() => {
    if (!saleId) return;
    const id = setInterval(async () => {
      const res = await fetch(`/api/sales/${saleId}/payment`);
      const data = await res.json();
      setStatus(data.status);
    }, 3000);
    return () => clearInterval(id);
  }, [saleId]);

  return status;
}
```

**The dependency array:**

| Array | Effect runs |
| --- | --- |
| None: `useEffect(fn)` | After every render (rarely what you want) |
| Empty: `useEffect(fn, [])` | Once after mount (cleanup on unmount) |
| `[a, b]` | After mount and whenever `a` or `b` changes |

Include **every** value from the component that the effect uses. Do not lie to the linter; restructure the code instead.

In development, React Strict Mode runs effects **twice** on mount (mount → cleanup → mount) on purpose, to expose missing cleanups.

**You might not need an effect.** Effects are the most over-used hook. Do NOT use them to:

- **Calculate values from props/state** — compute during render (section 13.4).
- **Respond to a user event** — put the code in the event handler (`onClick` saves the sale; no effect watching a "submitted" flag).
- **Fetch data in Next.js** — use Server Components (section 19) or TanStack Query (section 15).
- **Reset state when a prop changes** — give the component a `key`: `<SaleEditor key={sale.id} />` remounts it fresh.

**Fetching in an effect** (when you must, in a plain React app) needs cleanup to avoid race conditions:

```tsx
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/medicines?search=${term}`, { signal: controller.signal })
    .then((r) => r.json())
    .then(setResults)
    .catch((e) => { if (e.name !== 'AbortError') setError(e.message); });
  return () => controller.abort();     // old request cancelled when term changes
}, [term]);
```

**`useLayoutEffect`** runs before the browser paints. Use it only to measure the DOM (tooltip position) and avoid a flicker.

**`useEffectEvent`** (React 19.2) wraps a function so an effect can call the **latest** version of it without listing it as a dependency, e.g. reading the current cart inside a long-lived subscription without re-subscribing whenever the cart changes.

## 14.4 `useRef` — a value that survives renders without causing them

**What it is:** A box `{ current: value }` that persists across renders. Changing `.current` does **not** re-render. Two main uses: **DOM access** and **mutable values** (timer IDs, previous values).

```tsx
function POSSearch() {
  const inputRef = useRef<HTMLInputElement>(null);
  const scanBuffer = useRef('');                 // mutable value, no re-render

  usePosShortcuts(() => inputRef.current?.focus());

  return <input ref={inputRef} placeholder="Search or scan barcode" />;
}
```

**Passing a ref to your own component (React 19):** `ref` is now a normal prop for function components. `forwardRef` is no longer needed in new code.

```tsx
function TextInput({ ref, ...props }: React.ComponentProps<'input'>) {
  return <input ref={ref} className="rounded-md border px-3 py-2" {...props} />;
}
```

`useImperativeHandle` lets a component expose a custom set of methods through a ref (`ref.current.clear()`), rarely needed.

## 14.5 `useContext` — sharing data without prop drilling

**What it is:** Context lets a parent make a value available to **every** component below it, without passing props through each level.

```tsx
// context/cart-context.tsx
'use client';
import { createContext, use, useState } from 'react';

interface CartContextValue {
  items: CartItem[];
  add: (m: Medicine) => void;
  remove: (id: number) => void;
  clear: () => void;
  total: number;
}

const CartContext = createContext<CartContextValue | null>(null);

export function CartProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<CartItem[]>([]);
  const total = items.reduce((s, i) => s + i.medicine.price * i.qty, 0);

  const value: CartContextValue = {
    items,
    total,
    add: (m) => setItems((prev) => addLine(prev, m)),
    remove: (id) => setItems((prev) => prev.filter((i) => i.medicine.id !== id)),
    clear: () => setItems([]),
  };

  return <CartContext value={value}>{children}</CartContext>;   // React 19: <Context> is the provider
}

// A custom hook so consumers get a clear error if the provider is missing
export function useCart() {
  const ctx = use(CartContext);               // useContext(CartContext) also works
  if (!ctx) throw new Error('useCart must be used inside <CartProvider>');
  return ctx;
}
```

```tsx
// Anywhere below the provider
function CartCount() {
  const { items } = useCart();
  return <span>{items.length}</span>;
}
```

Every consumer re-renders when the context value changes. Split contexts that change at different speeds (theme vs cart), or use a store like Zustand for frequently-changing data (section 15).

## 14.6 `useReducer` — state with rules

**What it is:** For state with many related updates, put all the update logic in one **reducer** function: `(state, action) => newState`. Components only **dispatch** actions describing what happened.

```tsx
type CartAction =
  | { type: 'add'; medicine: Medicine }
  | { type: 'setQty'; id: number; qty: number }
  | { type: 'remove'; id: number }
  | { type: 'clear' };

function cartReducer(state: CartItem[], action: CartAction): CartItem[] {
  switch (action.type) {
    case 'add': {
      const found = state.find((i) => i.medicine.id === action.medicine.id);
      if (found && found.qty >= action.medicine.stock) return state;   // business rule in ONE place
      return found
        ? state.map((i) => (i === found ? { ...i, qty: i.qty + 1 } : i))
        : [...state, { medicine: action.medicine, qty: 1 }];
    }
    case 'setQty':
      return state.map((i) => (i.medicine.id === action.id ? { ...i, qty: Math.max(1, action.qty) } : i));
    case 'remove':
      return state.filter((i) => i.medicine.id !== action.id);
    case 'clear':
      return [];
  }
}

const [cart, dispatch] = useReducer(cartReducer, []);
dispatch({ type: 'add', medicine: amox });
```

Reducers are pure functions, so they are easy to unit-test without React.

## 14.7 `useMemo` and `useCallback` — caching between renders

- **`useMemo(fn, deps)`** caches a **calculated value**.
- **`useCallback(fn, deps)`** caches a **function** (same reference across renders).

```tsx
const lowStock = useMemo(
  () => medicines.filter((m) => m.stock < m.reorderLevel).toSorted((a, b) => a.stock - b.stock),
  [medicines],
);

const handleAdd = useCallback((m: Medicine) => dispatch({ type: 'add', medicine: m }), []);
```

Use them only when there is a measured problem: a genuinely expensive calculation, or a value passed to a `memo()` child or an effect dependency. With the **React Compiler** enabled (section 16), React memoises automatically and you rarely write these by hand.

## 14.8 Other built-in hooks

| Hook | What it does | PharmaCare use |
| --- | --- | --- |
| `useId()` | Unique, SSR-safe id | Link `<label htmlFor={id}>` to `<input id={id}>` |
| `useTransition()` | Mark an update as non-urgent; get `isPending` | Filtering 5,000 medicines without freezing typing |
| `useDeferredValue(v)` | A lagging copy of a value | Search results lag behind the input smoothly |
| `useOptimistic()` | Show the expected result before the server confirms | Cart line appears instantly while saving |
| `useActionState()` | State + pending flag for a form action | Server Action form with errors (section 20) |
| `useFormStatus()` (react-dom) | Pending state of the parent `<form>` | Disable the submit button while saving |
| `use(promise \| context)` | Read a promise (suspends) or context; allowed in conditions | Unwrap data passed from a Server Component |
| `useSyncExternalStore()` | Subscribe to an external store | `navigator.onLine` status |
| `useDebugValue()` | Label custom hooks in DevTools |  |

```tsx
const [isPending, startTransition] = useTransition();
function onSearch(term: string) {
  setTerm(term);                                   // urgent: input updates now
  startTransition(() => setFilter(term));          // non-urgent: big list can lag
}
```

## 14.9 Custom hooks

**What it is:** Your own function starting with `use` that combines other hooks. It shares **logic**, not state: each component calling it gets its own copy.

```tsx
// hooks/use-debounce.ts
export function useDebounce<T>(value: T, delay = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(t);
  }, [value, delay]);
  return debounced;
}

// hooks/use-local-storage.ts
export function useLocalStorage<T>(key: string, initial: T) {
  const [value, setValue] = useState<T>(() => {
    if (typeof window === 'undefined') return initial;     // safe during server rendering
    try { return JSON.parse(localStorage.getItem(key) ?? '') as T; } catch { return initial; }
  });
  useEffect(() => { localStorage.setItem(key, JSON.stringify(value)); }, [key, value]);
  return [value, setValue] as const;
}

// hooks/use-online.ts
export function useOnline() {
  return useSyncExternalStore(
    (cb) => { addEventListener('online', cb); addEventListener('offline', cb); return () => { removeEventListener('online', cb); removeEventListener('offline', cb); }; },
    () => navigator.onLine,
    () => true,                                            // server snapshot
  );
}
```

```tsx
const [term, setTerm] = useState('');
const debouncedTerm = useDebounce(term, 300);
const online = useOnline();
```

**Hooks best practices:** keep components small; extract logic into custom hooks with clear names (`useCart`, `useMedicineSearch`); keep effects minimal and always clean up; prefer derived values over synced state; follow the rules of hooks.
