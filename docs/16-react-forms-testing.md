# 16. Forms, UI libraries, performance and testing in React

## 16.1 Forms with React Hook Form + Zod

**What it is:** **React Hook Form** (RHF) manages form values, validation and errors with very few re-renders (it uses uncontrolled inputs). **Zod** defines the rules once (section 9.9). Together they are the standard for serious forms. Older alternatives: Formik, Final Form.

```bash
npm install react-hook-form zod @hookform/resolvers
```

```tsx
// schemas/batch.ts — shared by the form AND the server
import { z } from 'zod';

export const batchSchema = z.object({
  medicineId: z.coerce.number().int().positive('Choose a medicine'),
  batchNo: z.string().trim().toUpperCase().regex(/^[A-Z]{3}-\d{4}-\d{2}$/, 'Format: AMX-2026-01'),
  quantity: z.coerce.number().int().min(1, 'At least 1'),
  costPrice: z.coerce.number().int().min(0),
  expiresAt: z.coerce.date().refine((d) => d > new Date(), 'Batch has already expired'),
});
export type BatchInput = z.infer<typeof batchSchema>;
```

```tsx
// components/receive-stock-form.tsx
'use client';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { batchSchema, type BatchInput } from '@/schemas/batch';

export function ReceiveStockForm({ medicines }: { medicines: Medicine[] }) {
  const {
    register,
    handleSubmit,
    reset,
    setError,
    formState: { errors, isSubmitting },
  } = useForm({ resolver: zodResolver(batchSchema) });

  async function onSubmit(data: BatchInput) {
    try {
      await api('/batches', { method: 'POST', body: data });
      reset();
    } catch (e) {
      if (e instanceof ApiError && e.errors.batch_no) {
        setError('batchNo', { message: e.errors.batch_no[0] });   // show the server's error on the field
      }
    }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="grid gap-4" noValidate>
      <Field label="Medicine" error={errors.medicineId?.message}>
        <select {...register('medicineId')}>
          <option value="">Select…</option>
          {medicines.map((m) => <option key={m.id} value={m.id}>{m.name}</option>)}
        </select>
      </Field>
      <Field label="Batch number" error={errors.batchNo?.message}>
        <input {...register('batchNo')} />
      </Field>
      <Field label="Quantity" error={errors.quantity?.message}>
        <input type="number" {...register('quantity')} />
      </Field>
      <Field label="Expiry date" error={errors.expiresAt?.message}>
        <input type="date" {...register('expiresAt')} />
      </Field>
      <button disabled={isSubmitting}>{isSubmitting ? 'Saving…' : 'Receive stock'}</button>
    </form>
  );
}

function Field({ label, error, children }: { label: string; error?: string; children: React.ReactNode }) {
  return (
    <label className="grid gap-1 text-sm">
      <span className="font-medium">{label}</span>
      {children}
      {error && <span role="alert" className="text-red-600">{error}</span>}
    </label>
  );
}
```

Other RHF tools: `watch('quantity')` (read live values), `useFieldArray` (dynamic rows — the lines of a purchase order), `Controller` (wrap custom inputs like date pickers), `defaultValues` (edit forms).

## 16.2 Component libraries

| Library | Type | Notes |
| --- | --- | --- |
| **shadcn/ui** | Copy-in components built on Radix + Tailwind | You own the code; the default for Next.js + Tailwind |
| **Radix UI** | Headless primitives | Accessible dialogs, dropdowns, popovers; you style |
| **React Aria** (Adobe) | Headless hooks/components | Best-in-class accessibility and i18n |
| **Ark UI** | Headless, state-machine based | Works across React, Vue, Solid |
| **Headless UI** | Headless, by the Tailwind team | Small set of components |
| **Material UI (MUI)** | Fully styled (Material Design) | Fast admin panels; heavier |
| **Chakra UI**, **Mantine** | Styled with theming | Batteries included |

**shadcn/ui quick start:**

```bash
npx shadcn@latest init                    # sets up components.json, cn(), CSS variables
npx shadcn@latest add button dialog input table sonner
```

```tsx
import { Button } from '@/components/ui/button';
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogTrigger } from '@/components/ui/dialog';

<Dialog>
  <DialogTrigger asChild><Button variant="outline">Void sale</Button></DialogTrigger>
  <DialogContent>
    <DialogHeader><DialogTitle>Void sale RCP-00012?</DialogTitle></DialogHeader>
    <Button variant="destructive" onClick={voidSale}>Yes, void it</Button>
  </DialogContent>
</Dialog>
```

The components land in `components/ui/`, so you can edit them freely.

## 16.3 Animation

| Tool | Use for |
| --- | --- |
| Tailwind/CSS transitions | Hover, focus, simple fades (`transition`, `animate-*`) — cheapest |
| **Motion** (formerly Framer Motion) | Enter/exit animations, layout animations, gestures |
| **React Spring** | Physics-based springs |
| **GSAP** | Complex timelines, marketing pages |

```tsx
import { motion, AnimatePresence } from 'motion/react';

<AnimatePresence>
  {items.map((i) => (
    <motion.li key={i.medicine.id} layout initial={{ opacity: 0, y: -8 }} animate={{ opacity: 1, y: 0 }} exit={{ opacity: 0, x: 40 }}>
      {i.medicine.name} × {i.qty}
    </motion.li>
  ))}
</AnimatePresence>
```

Respect users who prefer less motion: Tailwind's `motion-reduce:` variant or Motion's `useReducedMotion()`.

## 16.4 React performance

**First, understand re-renders:** a component re-renders when its state changes, its parent re-renders, or a context it uses changes. Re-renders are usually cheap; only optimise what you have **measured** with the React DevTools **Profiler**.

| Technique | What it does |
| --- | --- |
| **React Compiler** | Build-time tool that memoises components and values automatically. Turn it on and stop hand-writing most `useMemo`/`useCallback`/`memo` |
| `memo(Component)` | Skip re-rendering a child when its props are unchanged |
| Keep state close | State in a small leaf component re-renders only that leaf |
| Pass children | `<Layout><Expensive /></Layout>`: children do not re-render when Layout's state changes |
| Virtualise long lists | Render only visible rows: TanStack Virtual, react-window (5,000 medicines) |
| `lazy()` + Suspense | Load heavy components (charts, PDF viewer) only when shown |
| `useTransition` / `useDeferredValue` | Keep typing responsive during heavy updates |
| Stable keys | Avoid remounting list items |

```ts
// next.config.ts — enable the React Compiler in Next.js
const nextConfig = { reactCompiler: true };
export default nextConfig;
// npm install -D babel-plugin-react-compiler
```

## 16.5 Testing React components

**What it is:** **Vitest** (or Jest) runs the tests; **React Testing Library** (RTL) renders components and interacts with them **the way a user does** — finding things by visible text and role, not by internal details. Full setup and end-to-end testing in section 26.

```bash
npm install -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/user-event @testing-library/jest-dom
```

```tsx
// components/quantity-picker.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect } from 'vitest';
import { QuantityPicker } from './quantity-picker';

describe('QuantityPicker', () => {
  it('increases quantity but not above stock', async () => {
    const user = userEvent.setup();
    render(<QuantityPicker max={2} />);

    const plus = screen.getByRole('button', { name: '+' });
    await user.click(plus);
    expect(screen.getByText('2')).toBeInTheDocument();
    expect(plus).toBeDisabled();
  });
});
```

**Query priority** (most to least preferred): `getByRole` → `getByLabelText` → `getByPlaceholderText` → `getByText` → `getByTestId`. `getBy` throws if missing, `queryBy` returns null (for "should not exist"), `findBy` waits (async).

## 16.6 TypeScript patterns for React

```tsx
// Props of a native element + your own
type InputProps = React.ComponentProps<'input'> & { label: string; error?: string };

// Children
type Props = { children: React.ReactNode };

// Events
const onChange = (e: React.ChangeEvent<HTMLSelectElement>) => {};

// Generic component: a typed data table
function DataTable<T>({ rows, columns }: {
  rows: T[];
  columns: { header: string; cell: (row: T) => React.ReactNode }[];
}) {
  return (
    <table>
      <thead><tr>{columns.map((c) => <th key={c.header}>{c.header}</th>)}</tr></thead>
      <tbody>{rows.map((r, i) => <tr key={i}>{columns.map((c) => <td key={c.header}>{c.cell(r)}</td>)}</tr>)}</tbody>
    </table>
  );
}

<DataTable rows={medicines} columns={[
  { header: 'Name', cell: (m) => m.name },          // m is typed as Medicine
  { header: 'Stock', cell: (m) => <StockBadge stock={m.stock} /> },
]} />
```

For large data tables with sorting, filtering and pagination, use **TanStack Table** (headless) with shadcn/ui's table components.
