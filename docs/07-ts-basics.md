# 7. TypeScript basics

## 7.1 What TypeScript is and why

**What it is:** TypeScript (TS) is JavaScript plus **types**. You describe what shape your data has ("price is a number", "a sale has items"), and the TypeScript compiler checks your code **before it runs**. TS is then converted ("compiled") to plain JS, so browsers never see the types.

**Why it matters:**

```js
// JavaScript: this bug reaches production
function total(items) {
  return items.reduce((s, i) => s + i.price * i.quantity, 0);  // the field is `qty`, not `quantity`
}
// → NaN on every receipt, discovered by a customer
```

```ts
// TypeScript: red underline in the editor, the second you type it
function total(items: CartLine[]): number {
  return items.reduce((s, i) => s + i.price * i.quantity, 0);
  //                                             ~~~~~~~~ Property 'quantity' does not exist on type 'CartLine'
}
```

You also get autocomplete for every field, safe renaming across files, and self-documenting code.

|  | JavaScript | TypeScript |
| --- | --- | --- |
| Types | Checked at runtime (when it crashes) | Checked at compile time (in the editor) |
| Files | `.js`, `.jsx` | `.ts`, `.tsx` |
| Runs in browser? | Directly | After compiling to JS |
| Valid JS? | — | All JS is valid TS; TS adds types on top |

**TS and JS interoperability:** you can mix `.js` and `.ts` files, import JS libraries into TS, and adopt TS file by file. Most npm packages ship type definitions.

## 7.2 Installation, running and configuration

```bash
npm install -D typescript          # add to a project
npx tsc --init                     # create tsconfig.json
npx tsc                            # compile the project (reports type errors)
npx tsc --noEmit                   # type-check only, emit nothing (common in CI)
npx tsc --watch                    # re-check on every save

npx tsx script.ts                  # run a .ts file directly (modern replacement for ts-node)
node script.ts                     # recent Node.js versions can strip types and run .ts directly
```

Next.js and Vite set TypeScript up for you: create the project with TypeScript selected and you are done. The **TS Playground** (typescriptlang.org/play) is the fastest place to experiment.

**`tsconfig.json` — the settings that matter:**

```json5
{
  "compilerOptions": {
    "target": "ES2022",              // which JS version to output
    "lib": ["dom", "dom.iterable", "esnext"],   // which built-in types exist (DOM for browsers)
    "module": "esnext",
    "moduleResolution": "bundler",   // how imports are found (bundler = Vite/Next.js)
    "jsx": "preserve",               // leave JSX for the framework to handle
    "strict": true,                  // ALWAYS on: enables all the important checks
    "noUncheckedIndexedAccess": true,// arr[0] might be undefined — catches real bugs
    "noEmit": true,                  // the bundler emits JS, tsc only checks
    "skipLibCheck": true,            // don't type-check node_modules (faster)
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }  // import from "@/lib/api" instead of "../../../lib/api"
  },
  "include": ["**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
```

Rule: keep `"strict": true`. Turning it off removes most of the value of TypeScript.

## 7.3 Basic types and annotations

```ts
// Primitives
let name: string = 'Amoxicillin';
let price: number = 500;
let inStock: boolean = true;
let big: bigint = 10n;
let id: symbol = Symbol('id');
let nothing: null = null;
let notSet: undefined = undefined;

// Arrays
let prices: number[] = [300, 500];
let names: Array<string> = ['Paracetamol'];   // same thing, generic syntax

// Tuples: fixed length, each position has its own type
let line: [medicineId: number, qty: number] = [7, 2];
const [medicineId, qty] = line;

// Objects (inline type)
let medicine: { id: number; name: string; price: number; imageUrl?: string } = {
  id: 7, name: 'Amoxicillin', price: 500,           // imageUrl is optional (?)
};

// Readonly
const VAT_RATES: readonly number[] = [0, 18];
// VAT_RATES.push(5);   // Error: readonly
```

## 7.4 Type inference

**What it is:** TS works out types itself from the value. You do not need to annotate everything.

```ts
let stock = 120;                 // inferred: number
const method = 'cash';           // inferred: the literal type 'cash' (const never changes)
const meds = [{ id: 1, price: 300 }];  // inferred: { id: number; price: number }[]
const total = meds.reduce((s, m) => s + m.price, 0);  // number

// stock = 'many';               // Error: Type 'string' is not assignable to type 'number'
```

Rule of thumb: **annotate function parameters and exported/public APIs; let TS infer local variables and return values** (annotate return types on important functions for clarity).

## 7.5 `any`, `unknown`, `never`, `void`, `object`

| Type | Meaning | Use |
| --- | --- | --- |
| `any` | "Turn off type checking for this value" | Avoid. It spreads silently and hides bugs |
| `unknown` | "Could be anything; check before use" | Data from outside: API responses, `JSON.parse`, `catch (e)` |
| `never` | "This can never happen" | Functions that always throw; exhaustive checks |
| `void` | "Returns nothing useful" | Return type of functions like `console.log` |
| `object` | Any non-primitive | Rare; prefer a specific shape |

```ts
function parseSale(json: string) {
  const data: unknown = JSON.parse(json);
  // data.total;                           // Error: 'data' is of type 'unknown'
  if (typeof data === 'object' && data !== null && 'total' in data) {
    return data.total;                     // allowed after checking
  }
}

function fail(message: string): never {
  throw new Error(message);
}

function logSale(id: number): void {
  console.log('sale', id);
}

try { /* ... */ } catch (e) {               // e is `unknown` in strict mode
  const message = e instanceof Error ? e.message : String(e);
}
```

**Escape hatches** (use rarely, they silence the compiler):

```ts
const input = document.getElementById('search') as HTMLInputElement;  // type assertion: "trust me"
const el = document.getElementById('total')!;    // non-null assertion: "this is not null"
const x = value as any;                           // as any: turns checking off entirely
```

## 7.6 Literal types and `as const`

**What it is:** A literal type allows only exact values. Combined with unions (section 8), it replaces magic strings.

```ts
let method: 'cash' | 'mpesa' | 'airtel' | 'card';
method = 'mpesa';     // OK
// method = 'paypal'; // Error

// as const: make an object/array deeply readonly with literal types
const ROLES = ['admin', 'pharmacist', 'cashier'] as const;
type Role = (typeof ROLES)[number];   // 'admin' | 'pharmacist' | 'cashier'

const STATUS_COLORS = { paid: 'green', pending: 'orange', voided: 'red' } as const;
```

## 7.7 Enums

**What it is:** A named set of constants.

```ts
enum SaleStatus {
  Pending = 'pending',
  Paid = 'paid',
  Voided = 'voided',
}

let status: SaleStatus = SaleStatus.Paid;
```

Enums generate extra JS code and behave differently from the rest of TS. Most modern codebases prefer a **union of string literals** instead: `type SaleStatus = 'pending' | 'paid' | 'voided'`. It is lighter and works the same in editors.

## 7.8 Running checks: TS in your workflow

- **Editor** (VS Code) shows errors as you type.
- **`tsc --noEmit`** in CI blocks merging broken types.
- **Next.js** runs a type check during `next build`.
- **ESLint with `typescript-eslint`** adds TS-aware lint rules (e.g. no floating promises). **Prettier** handles formatting. See section 25.
