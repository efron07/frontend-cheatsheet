# 12. React fundamentals

## 12.1 What React is and why frameworks exist

**What it is:** React is a JavaScript library for building user interfaces from **components** — reusable pieces like `<MedicineCard>`, `<Cart>`, `<SearchBar>`. You describe **what** the UI should look like for the current data; when the data changes, React updates the page for you.

**Why frontend frameworks exist:** with plain DOM code (section 6) you must update every affected element by hand each time data changes — the cart count, the total, the button state. In a POS screen with dozens of moving parts this becomes unmanageable. React lets you write:

```
UI = f(state)
```

The UI is a function of the state. Change the state; React works out the minimum DOM changes (this comparison step is called **reconciliation**, using a lightweight copy of the UI often called the virtual DOM).

**Alternatives:** Vue, Angular, Svelte, Solid. React has the largest ecosystem and is the base of Next.js and React Native (mobile apps with the same skills).

## 12.2 Creating a React app

| Tool | Use when |
| --- | --- |
| **Next.js** (`npx create-next-app@latest`) | Most real apps: routing, server rendering, SEO (section 17) |
| **Vite** (`npm create vite@latest pharmacare-pos -- --template react-ts`) | Pure client-side apps, learning, dashboards behind login |
| React Router framework mode, TanStack Start, Astro, Remix-style frameworks | Alternatives with their own strengths |

```bash
npm create vite@latest pharmacare-pos -- --template react-ts
cd pharmacare-pos
npm install
npm run dev          # http://localhost:5173
```

## 12.3 JSX

**What it is:** JSX is HTML-like syntax inside JavaScript. It is compiled into function calls that create React elements.

```tsx
const name = 'Amoxicillin';
const element = <h1 className="text-xl">Medicine: {name}</h1>;
// compiles to roughly: jsx('h1', { className: 'text-xl', children: ['Medicine: ', name] })
```

**JSX rules:**

- Return **one root element**. Use a fragment `<>...</>` to group without adding a DOM node.
- `class` → `className`, `for` → `htmlFor`, `onclick` → `onClick` (camelCase attributes).
- Close every tag: `<img />`, `<input />`.
- `{ }` holds any JavaScript **expression** (a value), not statements: `{price * qty}`, `{format(total)}`, `{isOpen ? 'Hide' : 'Show'}`.
- Inline styles are objects: `style={{ width: 120, backgroundColor: 'red' }}`.
- Comments: `{/* like this */}`.

## 12.4 Components

**What it is:** A component is a function that returns JSX. Its name **must start with a capital letter** (lowercase names are treated as HTML tags).

```tsx
// components/store-header.tsx
export function StoreHeader() {
  return (
    <header className="flex items-center justify-between border-b p-4">
      <span className="font-bold">PharmaCare</span>
      <nav className="flex gap-4">
        <a href="/medicines">Medicines</a>
        <a href="/branches">Branches</a>
      </nav>
    </header>
  );
}

// Use it like an HTML tag
export default function App() {
  return (
    <>
      <StoreHeader />
      <main>...</main>
    </>
  );
}
```

**Functional vs class components:** modern React uses **function components** with hooks. Class components (`class X extends React.Component` with `render()` and lifecycle methods like `componentDidMount`) are legacy; you will only see them in old code and in error boundaries.

**Pure rendering:** a component must return the same JSX for the same props and state, and must not change outside things (no fetching, no mutating variables) **while rendering**. Side effects belong in event handlers or effects (section 14).

## 12.5 Props

**What it is:** Props (properties) are the inputs you pass to a component, like function arguments. They flow **one way: parent → child**, and a child must never change its props.

```tsx
interface MedicineRowProps {
  medicine: Medicine;
  onAdd: (medicine: Medicine) => void;      // a function prop (callback)
  compact?: boolean;                         // optional
}

export function MedicineRow({ medicine, onAdd, compact = false }: MedicineRowProps) {
  return (
    <li className={compact ? 'py-1' : 'py-3'}>
      <span>{medicine.name}</span>
      <span>{medicine.price.toLocaleString()} TZS</span>
      <button onClick={() => onAdd(medicine)} disabled={medicine.stock === 0}>
        Add
      </button>
    </li>
  );
}

// Parent
<MedicineRow medicine={amox} onAdd={addToCart} compact />   // compact = true
```

**Spreading props** — pass extra HTML attributes through to the element:

```tsx
type ButtonProps = React.ComponentProps<'button'> & { variant?: 'primary' | 'outline' };

export function Button({ variant = 'primary', className, ...rest }: ButtonProps) {
  return <button className={cn(buttonVariants({ variant }), className)} {...rest} />;
}

<Button type="submit" disabled={pending} onClick={save}>Save</Button>
```

## 12.6 Children and composition

**What it is:** Whatever you put between a component's tags arrives as the `children` prop. This lets you build wrappers that do not care what is inside.

```tsx
export function Card({ title, children, footer }: {
  title: string;
  children: React.ReactNode;     // anything renderable: text, elements, arrays, null
  footer?: React.ReactNode;      // a "slot" prop
}) {
  return (
    <section className="rounded-xl border bg-white p-4">
      <h2 className="mb-3 font-semibold">{title}</h2>
      {children}
      {footer && <div className="mt-4 border-t pt-3">{footer}</div>}
    </section>
  );
}

<Card title="Low stock" footer={<a href="/stock">View all</a>}>
  <LowStockList />
</Card>
```

## 12.7 Rendering lists and keys

```tsx
export function MedicineList({ medicines }: { medicines: Medicine[] }) {
  if (medicines.length === 0) return <p className="text-gray-500">No medicines found.</p>;

  return (
    <ul className="divide-y">
      {medicines.map((m) => (
        <MedicineRow key={m.id} medicine={m} onAdd={addToCart} />
      ))}
    </ul>
  );
}
```

**What a `key` is:** a stable, unique identifier for each item among its siblings. React uses it to match items between renders, so it can move, add or remove the right ones. Use the record's id. **Do not use the array index** when the list can be reordered, filtered or edited: inputs and state end up attached to the wrong row.

## 12.8 Conditional rendering

```tsx
function CartSummary({ items, isLoading, error }: Props) {
  if (isLoading) return <Spinner />;                      // early return
  if (error) return <ErrorMessage message={error} />;

  return (
    <div>
      {items.length > 0 ? <CartTable items={items} /> : <EmptyCart />}   {/* ternary */}
      {items.some((i) => i.medicine.requiresPrescription) && (          // && for show/hide
        <p className="text-blue-700">Prescription required for some items</p>
      )}
    </div>
  );
}
```

**Watch out with `&&` and numbers:** `{items.length && <List />}` renders a literal `0` when the list is empty. Write `{items.length > 0 && <List />}`.

Returning `null` renders nothing.
