# 8. TypeScript: shaping data

## 8.1 Interfaces and type aliases

**What it is:** Two ways to give a shape a name so you can reuse it.

```ts
// interface: describes an object shape
interface Medicine {
  readonly id: number;          // cannot be changed after creation
  name: string;
  price: number;
  requiresPrescription: boolean;
  imageUrl?: string;            // optional
  [key: string]: unknown;       // index signature: allow extra keys (use sparingly)
}

// type alias: a name for ANY type — objects, unions, tuples, functions
type MedicineId = number;
type PaymentMethod = 'cash' | 'mpesa' | 'airtel' | 'card';
type Money = number;
type CartLine = { medicineId: MedicineId; qty: number; price: Money };
type Formatter = (amount: Money) => string;
```

**Extending:**

```ts
interface Product { id: number; name: string; price: number }
interface Medicine extends Product { requiresPrescription: boolean }

type Product2 = { id: number; name: string };
type Medicine2 = Product2 & { requiresPrescription: boolean };   // intersection
```

|  | `interface` | `type` |
| --- | --- | --- |
| Object shapes | Yes | Yes |
| Unions, tuples, primitives, functions | No | Yes |
| Extend | `extends` | `&` (intersection) |
| Declaration merging (same name twice merges) | Yes | No (error) |

Practical rule: use `interface` for object shapes (especially public APIs and props) and `type` for unions and everything else. Consistency matters more than the choice.

## 8.2 Union types

**What it is:** A value that can be one of several types: `A | B`.

```ts
type Id = number | string;

function findMedicine(id: Id) { /* ... */ }
findMedicine(7);
findMedicine('amoxicillin-500mg');
```

**Discriminated unions** — the most useful pattern in TS. Each variant has a common literal field (the "tag") that tells them apart:

```ts
type Payment =
  | { method: 'cash'; tendered: number }
  | { method: 'mpesa'; phone: string; transactionId?: string }
  | { method: 'card'; last4: string };

function describePayment(p: Payment): string {
  switch (p.method) {
    case 'cash':  return `Cash, tendered ${p.tendered}`;     // TS knows p.tendered exists here
    case 'mpesa': return `M-Pesa from ${p.phone}`;
    case 'card':  return `Card ****${p.last4}`;
    default: {
      const _exhaustive: never = p;   // compile error if a new method is added but not handled
      return _exhaustive;
    }
  }
}

// The same pattern for request state in React
type LoadState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };
```

## 8.3 Intersection types

**What it is:** `A & B` — a value with all the properties of both.

```ts
type Timestamps = { createdAt: string; updatedAt: string };
type SaleRecord = Sale & Timestamps;
```

## 8.4 Narrowing and type guards

**What it is:** Inside an `if`, TS narrows a broad type to a specific one based on your check.

```ts
function formatId(id: number | string) {
  if (typeof id === 'string') return id.toUpperCase();   // here: string
  return id.toFixed(0);                                   // here: number
}

// truthiness narrowing
function supplierName(m: { supplier?: { name: string } }) {
  if (!m.supplier) return 'Unknown';
  return m.supplier.name;                   // supplier is defined here
}

// `in` narrowing
if ('phone' in payment) { payment.phone }

// instanceof narrowing
if (error instanceof ApiError) { error.status }

// equality narrowing
if (payment.method === 'mpesa') { payment.phone }
```

**Custom type guards (type predicates)** — a function that returns `value is Type`:

```ts
function isMedicine(x: unknown): x is Medicine {
  return typeof x === 'object' && x !== null && 'id' in x && 'price' in x;
}

const items: unknown[] = await res.json();
const medicines = items.filter(isMedicine);   // Medicine[]

// Assertion function: throws if not true, narrows after the call
function assertDefined<T>(v: T | null | undefined, msg = 'Missing value'): asserts v is T {
  if (v == null) throw new Error(msg);
}
```

**Truthiness and equality in TS:** narrowing follows JS rules from section 1, so `if (stock)` excludes `0`. Use `stock !== undefined` when zero is valid.

## 8.5 Typing functions

```ts
// Parameters and return type
function lineTotal(price: number, qty: number): number {
  return price * qty;
}

// Optional and default parameters
function applyDiscount(total: number, percent = 0, reason?: string): number { /* ... */ }

// Rest parameters
function sum(...amounts: number[]): number { return amounts.reduce((a, b) => a + b, 0); }

// Arrow function type
const toTZS: (amount: number) => string = (a) => `${a.toLocaleString()} TZS`;

// Object parameter (the most common style)
interface CreateSaleInput {
  items: CartLine[];
  paymentMethod: PaymentMethod;
  customerId?: number;
}
async function createSale({ items, paymentMethod, customerId }: CreateSaleInput): Promise<Sale> { /* ... */ }

// Callbacks
function onScan(handler: (barcode: string) => void) { /* ... */ }
```

**Function overloading** — several signatures for one function, when the return type depends on the input:

```ts
function getMedicine(id: number): Promise<Medicine>;
function getMedicine(ids: number[]): Promise<Medicine[]>;
function getMedicine(idOrIds: number | number[]) {
  return Array.isArray(idOrIds) ? fetchMany(idOrIds) : fetchOne(idOrIds);
}
```

## 8.6 Classes in TypeScript

```ts
abstract class PaymentGateway {                // abstract: cannot be created directly
  constructor(protected readonly apiKey: string) {}   // parameter property: declares + assigns

  abstract charge(phone: string, amount: number): Promise<string>;   // subclasses MUST implement

  protected log(msg: string): void {           // protected: this class and subclasses
    console.log(`[${this.constructor.name}] ${msg}`);
  }
}

class MpesaGateway extends PaymentGateway {
  private retries = 3;                          // private: this class only (TS-level)

  async charge(phone: string, amount: number): Promise<string> {   // method overriding
    this.log(`Charging ${amount} to ${phone}`);
    return 'MP' + Date.now();
  }
}

interface Printable { print(): void }
class Receipt implements Printable {          // implements: must satisfy the interface
  constructor(public sale: Sale) {}
  print() { window.print(); }
}
```

| Modifier | Visible to |
| --- | --- |
| `public` (default) | Everyone |
| `protected` | The class and its subclasses |
| `private` | Only the class (compile-time check). `#field` is the JS runtime-private version |
| `readonly` | Can be set only in the constructor |

**Inheritance vs polymorphism:** inheritance is "MpesaGateway **is a** PaymentGateway". Polymorphism is code that works with any `PaymentGateway` without knowing which one: `async function pay(g: PaymentGateway) { await g.charge(...) }`.

**Constructor overloading** works like function overloading: declare several `constructor(...)` signatures, then one implementation.

**Decorators** (`@something` above a class or method) add behaviour to classes. They are common in Angular and NestJS; you rarely need them in React or Next.js.

## 8.7 Type compatibility

**What it is:** TS is **structurally typed**: if an object has the required properties, it fits, whatever its name. This is sometimes called "duck typing".

```ts
interface HasPrice { price: number }
const amox = { id: 7, name: 'Amoxicillin', price: 500 };
function showPrice(x: HasPrice) { /* ... */ }
showPrice(amox);    // OK: it has `price`, extra fields are fine when passing a variable
// showPrice({ price: 1, name: 'x' });   // Error: object literals are checked for extra keys
```

## 8.8 Hybrid types and recursive types

```ts
// Recursive: a category tree
interface CategoryNode {
  id: number;
  name: string;
  children: CategoryNode[];
}

// Hybrid: a function that also has properties
interface Counter {
  (): number;           // callable
  reset(): void;        // and has a method
}
```
