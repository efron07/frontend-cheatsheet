# 13. State and events

## 13.1 Props vs state

|  | Props | State |
| --- | --- | --- |
| Comes from | The parent | Inside the component |
| Can the component change it? | No (read-only) | Yes, with its setter |
| Changing it | Parent re-renders with new props | Triggers a re-render of this component and its children |
| Example | `medicine` passed to `<MedicineRow>` | The search text, the cart, "is the modal open?" |

## 13.2 `useState`

**What it is:** A hook that gives a component memory between renders. It returns the current value and a function to change it.

```tsx
import { useState } from 'react';

export function QuantityPicker({ max }: { max: number }) {
  const [qty, setQty] = useState(1);        // initial value 1

  return (
    <div className="flex items-center gap-2">
      <button onClick={() => setQty((q) => Math.max(1, q - 1))}>−</button>
      <span className="w-8 text-center">{qty}</span>
      <button onClick={() => setQty((q) => Math.min(max, q + 1))} disabled={qty >= max}>+</button>
    </div>
  );
}
```

**How state updates work — three rules that prevent most bugs:**

1. **Setting state schedules a re-render; it does not change the variable immediately.** `setQty(5); console.log(qty)` still logs the old value.
2. **Use the updater function when the new value depends on the old one:** `setQty((q) => q + 1)`. Calling `setQty(qty + 1)` three times in a row only adds 1, because `qty` is the same stale value each time.
3. **React batches updates:** several `set` calls in one event cause one re-render.

**Lazy initial state** — pass a function when the starting value is expensive:

```tsx
const [cart, setCart] = useState<CartItem[]>(() => JSON.parse(localStorage.getItem('cart') ?? '[]'));
```

## 13.3 Immutability: updating objects and arrays

**What it is:** React decides whether to re-render by checking if the state value is a **new object** (`Object.is`). If you change an object in place, React sees the same object and may not update. Always create a new object or array.

```tsx
const [cart, setCart] = useState<CartItem[]>([]);

// ADD (or increase qty if already in cart)
function addToCart(medicine: Medicine) {
  setCart((prev) => {
    const existing = prev.find((i) => i.medicine.id === medicine.id);
    if (existing) {
      return prev.map((i) => (i.medicine.id === medicine.id ? { ...i, qty: i.qty + 1 } : i));
    }
    return [...prev, { medicine, qty: 1 }];
  });
}

// UPDATE one item
function setQty(id: number, qty: number) {
  setCart((prev) => prev.map((i) => (i.medicine.id === id ? { ...i, qty } : i)));
}

// REMOVE
function removeItem(id: number) {
  setCart((prev) => prev.filter((i) => i.medicine.id !== id));
}

// CLEAR
setCart([]);

// WRONG — mutating
// cart.push({ medicine, qty: 1 }); setCart(cart);   // same array: React may skip the update
```

```tsx
// Objects
const [filters, setFilters] = useState({ search: '', category: 'all', inStockOnly: false });
setFilters((f) => ({ ...f, search: 'amox' }));

// Nested objects: copy every level you change
setSale((s) => ({ ...s, customer: { ...s.customer, phone: '0754000000' } }));
```

For deeply nested state, the **Immer** library (`useImmer`) lets you write mutating code that produces immutable updates.

## 13.4 Derived state: don't store what you can calculate

```tsx
// WRONG: total stored separately can go out of sync with the cart
const [total, setTotal] = useState(0);

// RIGHT: calculate it during render
const total = cart.reduce((sum, i) => sum + i.medicine.price * i.qty, 0);
const itemCount = cart.reduce((n, i) => n + i.qty, 0);
const needsPrescription = cart.some((i) => i.medicine.requiresPrescription);
```

## 13.5 Handling events

```tsx
export function SearchBar({ onSearch }: { onSearch: (term: string) => void }) {
  const [term, setTerm] = useState('');

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    setTerm(e.target.value);
  }

  function handleKeyDown(e: React.KeyboardEvent<HTMLInputElement>) {
    if (e.key === 'Enter') onSearch(term);
    if (e.key === 'Escape') setTerm('');
  }

  return (
    <input
      value={term}
      onChange={handleChange}
      onKeyDown={handleKeyDown}
      placeholder="Search medicine (F2)"
      className="w-full rounded-md border px-3 py-2"
    />
  );
}
```

- Pass the function, do not call it: `onClick={save}` not `onClick={save()}`. To pass arguments use an arrow: `onClick={() => removeItem(m.id)}`.
- Common events: `onClick`, `onChange`, `onSubmit`, `onKeyDown`, `onFocus`, `onBlur`, `onMouseEnter`, `onScroll`.
- React events are wrapped ("synthetic events") but have the same `preventDefault()` and `stopPropagation()`.
- Event types: `React.MouseEvent<HTMLButtonElement>`, `React.ChangeEvent<HTMLInputElement>`, `React.FormEvent<HTMLFormElement>`.

## 13.6 Forms: controlled vs uncontrolled

**Controlled input:** React state holds the value (`value` + `onChange`). Good for live validation and inputs that affect other UI.

```tsx
function ReceiveStockForm({ onSubmit }: { onSubmit: (data: NewBatch) => void }) {
  const [form, setForm] = useState({ batchNo: '', quantity: 1, expiresAt: '' });
  const [error, setError] = useState<string | null>(null);

  function update<K extends keyof typeof form>(key: K, value: (typeof form)[K]) {
    setForm((f) => ({ ...f, [key]: value }));
  }

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    if (new Date(form.expiresAt) <= new Date()) return setError('Batch is already expired');
    setError(null);
    onSubmit(form);
  }

  return (
    <form onSubmit={handleSubmit} className="space-y-3">
      <input value={form.batchNo} onChange={(e) => update('batchNo', e.target.value.toUpperCase())} />
      <input type="number" min={1} value={form.quantity} onChange={(e) => update('quantity', e.target.valueAsNumber)} />
      <input type="date" value={form.expiresAt} onChange={(e) => update('expiresAt', e.target.value)} />
      {error && <p role="alert" className="text-sm text-red-600">{error}</p>}
      <button type="submit">Receive stock</button>
    </form>
  );
}
```

**Uncontrolled input:** the DOM holds the value; you read it on submit with `FormData` (or a ref). Less code and re-renders. This is the style Next.js Server Actions use (section 20).

```tsx
function QuickSearch() {
  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const term = new FormData(e.currentTarget).get('q') as string;
    search(term);
  }
  return (
    <form onSubmit={handleSubmit}>
      <input name="q" defaultValue="" />   {/* defaultValue, not value */}
    </form>
  );
}
```

For bigger forms, use React Hook Form + Zod (section 16).

## 13.7 Lifting state up

**What it is:** When two components need the same state, move it up to their **closest common parent** and pass it down as props, together with functions to change it.

```tsx
// The search box and the results list both need `term`, so POSPage owns it
export function POSPage({ medicines }: { medicines: Medicine[] }) {
  const [term, setTerm] = useState('');
  const [cart, setCart] = useState<CartItem[]>([]);

  const results = medicines.filter((m) => m.name.toLowerCase().includes(term.toLowerCase()));

  return (
    <div className="grid lg:grid-cols-[1fr_380px]">
      <section>
        <SearchBar value={term} onChange={setTerm} />
        <MedicineList medicines={results} onAdd={(m) => setCart((c) => addLine(c, m))} />
      </section>
      <Cart items={cart} onChangeQty={(id, qty) => setCart((c) => changeQty(c, id, qty))} />
    </div>
  );
}
```

If passing props through many levels becomes painful ("prop drilling"), use **composition** or **context** (sections 14–15).

## 13.8 Where state should live: a quick guide

| Kind of state | Where | Tool |
| --- | --- | --- |
| One component's UI (open/closed, input text) | That component | `useState` |
| Shared by siblings | Closest common parent | Lift state up |
| App-wide UI (theme, logged-in user, cart) | Context or a store | `useContext`, Zustand |
| Server data (medicines, sales) | A server cache | Server Components / TanStack Query |
| Shareable/bookmarkable (search, filters, page) | The URL | `?search=amox&page=2` (`useSearchParams`) |
| Persist across reloads | Storage or the server | `localStorage`, database |
