# 9. TypeScript: generics and advanced types

## 9.1 Generics

**What it is:** A generic is a **type parameter** — a placeholder (`<T>`) that is filled in when the function or type is used. It lets one piece of code work with many types while staying fully typed.

**Simple picture:** a box that can hold medicines, sales or customers, and still knows exactly what is inside.

```ts
// Without generics: you lose the type
function firstAny(list: any[]): any { return list[0]; }

// With generics: the return type follows the input
function first<T>(list: T[]): T | undefined {
  return list[0];
}
const m = first(medicines);   // Medicine | undefined
const s = first(sales);       // Sale | undefined
```

**Generic types — typing API responses (Laravel shape):**

```ts
interface Paginated<T> {
  data: T[];
  meta: { current_page: number; last_page: number; per_page: number; total: number };
  links: { next: string | null; prev: string | null };
}

interface ApiResult<T> {
  data: T;
  message?: string;
}

async function apiGet<T>(path: string): Promise<T> {
  const res = await fetch(`${API}${path}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<T>;
}

const page = await apiGet<Paginated<Medicine>>('/medicines?page=1');
page.data[0]?.name;          // fully typed
```

**Generic constraints** — limit what `T` can be with `extends`:

```ts
function sortByPrice<T extends { price: number }>(items: T[]): T[] {
  return items.toSorted((a, b) => a.price - b.price);
}

function getField<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
getField(amox, 'price');     // number
// getField(amox, 'colour'); // Error: not a key of Medicine

// Default type parameter
interface ApiError<TErrors = Record<string, string[]>> { message: string; errors: TErrors }
```

## 9.2 `typeof`, `keyof` and indexed access

```ts
// typeof (in a type position): get the type of a value
const defaultFilters = { search: '', category: 'all', inStockOnly: false };
type Filters = typeof defaultFilters;   // { search: string; category: string; inStockOnly: boolean }

// keyof: union of an object type's keys
type MedicineKey = keyof Medicine;      // 'id' | 'name' | 'price' | ...

// Indexed access: the type of one property
type Price = Medicine['price'];         // number
type CategoryOf = Medicine['category']; // Category
type Item = Sale['items'][number];      // the type of one element of the items array
```

## 9.3 Utility types (built in)

| Utility | What it does | PharmaCare example |
| --- | --- | --- |
| `Partial<T>` | All properties optional | `Partial<Medicine>` for a PATCH update |
| `Required<T>` | All properties required | Ensure a draft is complete |
| `Readonly<T>` | All properties read-only | Frozen config |
| `Pick<T, K>` | Keep only some keys | `Pick<Medicine, 'id' \| 'name' \| 'price'>` for a list row |
| `Omit<T, K>` | Remove some keys | `Omit<Medicine, 'id'>` for a create form |
| `Record<K, V>` | Object with keys K and values V | `Record<PaymentMethod, number>` totals per method |
| `Exclude<U, X>` | Remove members from a union | `Exclude<PaymentMethod, 'card'>` |
| `Extract<U, X>` | Keep matching members | `Extract<PaymentMethod, 'mpesa' \| 'airtel'>` |
| `NonNullable<T>` | Remove `null` and `undefined` | Clean optional values |
| `ReturnType<F>` | Return type of a function | `ReturnType<typeof useCart>` |
| `Parameters<F>` | Parameter types as a tuple | Wrap an existing function |
| `Awaited<T>` | Unwrap a Promise | `Awaited<ReturnType<typeof getMedicines>>` |
| `InstanceType<C>` | Type of `new C()` | Instance of a class stored in a variable |

```ts
type NewMedicine = Omit<Medicine, 'id' | 'stock'>;
type MedicineUpdate = Partial<NewMedicine>;
type MedicineRow = Pick<Medicine, 'id' | 'name' | 'price' | 'stock'>;

const totalsByMethod: Record<PaymentMethod, number> = { cash: 0, mpesa: 0, airtel: 0, card: 0 };

type MedicinesResponse = Awaited<ReturnType<typeof getMedicines>>;
```

## 9.4 Mapped types

**What it is:** Build a new type by looping over the keys of another. This is how `Partial` and friends are made.

```ts
// Every field of a form becomes an error message (or undefined)
type FormErrors<T> = { [K in keyof T]?: string };
type MedicineFormErrors = FormErrors<NewMedicine>;
// { name?: string; price?: string; ... }

// Remove readonly with -readonly, make optional with ?
type Mutable<T> = { -readonly [K in keyof T]: T[K] };
```

## 9.5 Conditional types

**What it is:** "If T is X, then this type, else that type": `T extends X ? A : B`.

```ts
type IsArray<T> = T extends unknown[] ? true : false;
type A = IsArray<Medicine[]>;    // true

// infer: pull a type out of another
type ElementOf<T> = T extends (infer U)[] ? U : never;
type M = ElementOf<Medicine[]>;  // Medicine
```

## 9.6 Template literal types

```ts
type Branch = 'dodoma' | 'dar' | 'arusha';
type CacheKey = `medicines:${Branch}`;          // 'medicines:dodoma' | 'medicines:dar' | ...
type EventName = `on${Capitalize<'scan' | 'pay'>}`;   // 'onScan' | 'onPay'
```

## 9.7 The `satisfies` operator

**What it is:** Check that a value matches a type **without** widening it, so you keep the exact literal types.

```ts
const PAYMENT_LABELS = {
  cash: 'Cash',
  mpesa: 'M-Pesa',
  airtel: 'Airtel Money',
  card: 'Card',
} satisfies Record<PaymentMethod, string>;   // error if you forget a method or add a wrong key

PAYMENT_LABELS.mpesa;    // type is the literal 'M-Pesa', not just string
```

## 9.8 Modules, declaration files and ambient types

- **TypeScript modules** use the same `import`/`export` as JS. Export types with `export type { Medicine }` and import them with `import type { Medicine } from '@/types/pharmacy'` (removed from the JS output).
- **Declaration files** (`.d.ts`) contain only types, no code. Libraries ship them; for JS-only libraries install community types: `npm i -D @types/lodash`.
- **Ambient modules** declare types for things TS cannot see:

```ts
// types/global.d.ts
declare module '*.svg' {
  const src: string;
  export default src;
}

// Global augmentation: add a property to window
declare global {
  interface Window { dataLayer: unknown[] }
}

// Typing environment variables
declare namespace NodeJS {
  interface ProcessEnv {
    NEXT_PUBLIC_API_URL: string;
    DATABASE_URL: string;
  }
}
export {};
```

- **Namespaces** (`namespace X { }`) are an older way to group code. Use modules instead; you only meet namespaces in declaration files and namespace augmentation like the one above.

## 9.9 Runtime validation with Zod

**What it is:** TypeScript types disappear when the code runs, so TS cannot check data that arrives from outside (API responses, form input, URL params). **Zod** checks data at runtime **and** gives you the TypeScript type from the same definition. Alternatives: Valibot (smaller), Yup, ArkType.

```bash
npm install zod
```

```ts
import { z } from 'zod';

export const MedicineSchema = z.object({
  id: z.number().int(),
  name: z.string().min(2).max(150),
  price: z.number().int().nonnegative(),
  requiresPrescription: z.boolean(),
  stock: z.number().int().default(0),
  imageUrl: z.url().optional(),                  // Zod 4 top-level format helpers
});

export type Medicine = z.infer<typeof MedicineSchema>;   // the TS type, for free

// Validate an API response
const data = MedicineSchema.array().parse(await res.json());   // throws if the shape is wrong

// Validate form input without throwing
const NewBatchSchema = z.object({
  batchNo: z.string().trim().toUpperCase().regex(/^[A-Z]{3}-\d{4}-\d{2}$/, 'Format: AMX-2026-01'),
  quantity: z.coerce.number().int().min(1),             // coerce: "20" from a form → 20
  expiresAt: z.coerce.date().refine((d) => d > new Date(), 'Batch has already expired'),
});

const result = NewBatchSchema.safeParse(formValues);
if (!result.success) {
  console.log(z.flattenError(result.error).fieldErrors);  // { quantity: ['...'], ... }
} else {
  result.data;                                           // typed and cleaned
}
```

You will use one Zod schema in three places in Next.js: the form (React Hook Form), the Server Action, and the API response check.

## 9.10 Useful TypeScript packages

| Package | Purpose |
| --- | --- |
| `zod` / `valibot` | Runtime validation + inferred types |
| `typescript-eslint` | TS-aware linting |
| `tsx` | Run `.ts` files directly |
| `type-fest` | Extra utility types |
| `@types/*` | Types for JS-only libraries |
