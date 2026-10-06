# 3. Working with data

## 3.1 Objects

**What it is:** An object groups related values under named keys (properties). Most data you handle — a medicine, a sale, an API response — is an object.

```js
const medicine = {
  id: 7,
  name: 'Amoxicillin 500mg',
  price: 500,
  'requires-rx': true,           // keys with dashes need quotes
  category: { id: 1, name: 'Antibiotics' },
  describe() {                   // a method
    return `${this.name} — ${this.price} TZS`;
  },
};

medicine.name;                   // dot access
medicine['requires-rx'];         // bracket access (any string, or a variable)
const field = 'price';
medicine[field];                 // 500

medicine.stock = 120;            // add
medicine.price = 550;            // update
delete medicine['requires-rx'];  // remove

'stock' in medicine;             // true
Object.hasOwn(medicine, 'stock');// true (own property, not inherited)
medicine.supplier?.name;         // undefined instead of crashing
```

**Shorthand and computed keys:**

```js
const name = 'Ibuprofen', price = 800;
const med = { name, price };                 // same as { name: name, price: price }
const key = 'stock';
const withStock = { ...med, [key]: 40 };     // computed key
```

**Useful `Object` methods:**

```js
Object.keys(medicine);       // ['id', 'name', 'price', ...]
Object.values(medicine);     // [7, 'Amoxicillin 500mg', 500, ...]
Object.entries(medicine);    // [['id', 7], ['name', '...'], ...]
Object.fromEntries([['a', 1], ['b', 2]]);   // { a: 1, b: 2 }
Object.assign({}, a, b);     // merge (older way; prefer spread)
Object.freeze(config);       // make read-only (shallow)
structuredClone(sale);       // deep copy (nested objects too)
```

## 3.2 Arrays

**What it is:** An ordered list. Index starts at 0.

```js
const medicines = [
  { id: 1, name: 'Paracetamol 500mg', price: 300, stock: 120, category: 'Painkillers' },
  { id: 2, name: 'Amoxicillin 500mg', price: 500, stock: 8,   category: 'Antibiotics' },
  { id: 3, name: 'Ibuprofen 400mg',   price: 800, stock: 0,   category: 'Painkillers' },
  { id: 4, name: 'Metformin 500mg',   price: 450, stock: 60,  category: 'Antidiabetics' },
];

medicines.length;        // 4
medicines[0];            // first
medicines.at(-1);        // last
```

**The array methods you will use every day** (none of these change the original array):

```js
// map: transform every item → new array of the same length
const names = medicines.map((m) => m.name);

// filter: keep items that pass a test
const inStock = medicines.filter((m) => m.stock > 0);
const lowStock = medicines.filter((m) => m.stock > 0 && m.stock < 20);

// find / findIndex / findLast: first match (or undefined / -1)
const amox = medicines.find((m) => m.id === 2);

// some / every: yes/no questions
medicines.some((m) => m.stock === 0);    // true — is anything out of stock?
medicines.every((m) => m.price > 0);     // true

// includes: is a value present?
['cash', 'mpesa'].includes(method);

// reduce: boil the array down to ONE value
const stockValue = medicines.reduce((total, m) => total + m.price * m.stock, 0);

// group with reduce
const byCategory = medicines.reduce((groups, m) => {
  (groups[m.category] ??= []).push(m);
  return groups;
}, {});

// or the built-in (ES2024)
const grouped = Object.groupBy(medicines, (m) => m.category);

// slice: copy a part (start, end-exclusive)
medicines.slice(0, 2);

// flat / flatMap
[[1, 2], [3]].flat();                       // [1, 2, 3]
sales.flatMap((s) => s.items);              // all items of all sales in one list

// join
names.join(', ');

// Copying versions of mutating methods (ES2023)
const sorted = medicines.toSorted((a, b) => a.price - b.price);
const reversed = medicines.toReversed();
const updated = medicines.with(1, { ...medicines[1], stock: 50 });
```

**Chaining** — the real power:

```js
// Names of in-stock painkillers, cheapest first
const list = medicines
  .filter((m) => m.category === 'Painkillers' && m.stock > 0)
  .toSorted((a, b) => a.price - b.price)
  .map((m) => m.name);
```

**Mutating methods** (change the original — avoid on React state): `push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`, `fill`.

```js
const cart = [];
cart.push(item);           // add to end
cart.pop();                // remove from end
cart.unshift(item);        // add to start
cart.shift();              // remove from start
cart.splice(1, 1);         // remove 1 item at index 1
```

**Creating arrays:** `Array.from({ length: 5 }, (_, i) => i + 1)` → `[1,2,3,4,5]`; `Array.from(new Set(list))` removes duplicates; `Array.isArray(x)` checks.

## 3.3 Destructuring

**What it is:** Pull values out of objects and arrays into variables in one line.

```js
const { name, price, category: { name: categoryName }, stock = 0 } = medicine;
// renamed nested value; default if missing

const [first, second, ...others] = medicines;
const [, , third] = medicines;             // skip items

let a = 1, b = 2;
[a, b] = [b, a];                           // swap

// In function parameters (you will see this in every React component)
function MedicineRow({ name, price, stock = 0 }) { /* ... */ }
```

## 3.4 Spread and rest (`...`)

- **Spread** expands an array or object into its parts.
- **Rest** collects remaining parts into one array or object.

```js
// Spread: copy and extend
const updated = { ...medicine, price: 550 };          // new object, price changed
const moreMeds = [...medicines, newMedicine];        // new array with one more item
const merged = { ...defaults, ...userSettings };      // later keys win
Math.max(...medicines.map((m) => m.price));           // spread into arguments

// Rest: collect the rest
const { id, ...withoutId } = medicine;                // everything except id
```

Spread copies are **shallow**: nested objects are still shared. Use `structuredClone` for a deep copy.

## 3.5 Keyed collections: `Map`, `Set`, `WeakMap`, `WeakSet`

| Collection | What it holds | Use for |
| --- | --- | --- |
| `Map` | Key → value, keys of **any** type, keeps insertion order | Cart keyed by medicine id, caches |
| `Set` | Unique values | Selected IDs, removing duplicates |
| `WeakMap` | Object keys only; entries vanish when the key object is garbage-collected | Private data attached to objects |
| `WeakSet` | Objects only, weakly held | "Have I seen this object?" |

```js
const cart = new Map();
cart.set(2, { medicine: amox, qty: 1 });
cart.get(2).qty += 1;
cart.has(2);         // true
cart.delete(2);
cart.size;           // 0
for (const [id, line] of cart) { /* ... */ }

const selected = new Set([1, 2, 2, 3]);   // {1, 2, 3}
selected.add(4);
selected.has(2);    // true
const uniqueCategories = [...new Set(medicines.map((m) => m.category))];
```

**Indexed collections** = arrays and typed arrays (accessed by number). **Keyed collections** = Map, Set, WeakMap, WeakSet (accessed by key).

## 3.6 Strings and template literals

```js
const name = 'Amoxicillin 500mg';

`${name} costs ${price} TZS`;     // template literal: insert values
`Line 1
Line 2`;                           // multi-line

name.length;                       // 17
name.toUpperCase();
name.includes('500');              // true
name.startsWith('Amox');           // true
name.indexOf('500');               // 12
name.slice(0, 11);                 // "Amoxicillin"
name.split(' ');                   // ['Amoxicillin', '500mg']
name.replace('500mg', '250mg');
name.replaceAll('0', 'O');
'  hi  '.trim();                   // "hi"
'7'.padStart(4, '0');              // "0007" — receipt numbers
name.toLowerCase().includes(search.toLowerCase());   // case-insensitive search

// slug for URLs
const slug = name.toLowerCase().trim().replace(/[^a-z0-9]+/g, '-').replace(/(^-|-$)/g, '');
// "amoxicillin-500mg"
```

**Regular expressions** match text patterns: `/^0[67]\d{8}$/.test('0754123456')` checks a Tanzanian mobile number.

## 3.7 Numbers and maths

```js
Number.isInteger(500);         // true
Number.MAX_SAFE_INTEGER;       // 9007199254740991
(1234.567).toFixed(2);         // "1234.57" (a string)
Math.round(4.5);               // 5
Math.floor(4.9);  Math.ceil(4.1);
Math.min(3, 1, 2); Math.max(3, 1, 2);
Math.random();                 // 0 ≤ x < 1

// BigInt for integers beyond the safe range
const big = 9007199254740993n + 2n;
// cannot mix: 10n + 1 → TypeError; use 10n + BigInt(1)
```

**Format numbers and money for people with `Intl`:**

```js
new Intl.NumberFormat('en-TZ').format(1250000);   // "1,250,000"
new Intl.NumberFormat('en-TZ', { style: 'currency', currency: 'TZS', maximumFractionDigits: 0 }).format(1500);
// "TSh 1,500"
new Intl.NumberFormat('en', { notation: 'compact' }).format(1250000);   // "1.3M"
```

## 3.8 Dates

```js
const now = new Date();
const expiry = new Date('2027-03-31');          // ISO string
expiry.getFullYear();                            // 2027
expiry.toISOString();                            // "2027-03-31T00:00:00.000Z"

const daysLeft = Math.ceil((expiry - now) / (1000 * 60 * 60 * 24));

new Intl.DateTimeFormat('en-GB', { dateStyle: 'medium' }).format(expiry);   // "31 Mar 2027"
new Intl.RelativeTimeFormat('en', { numeric: 'auto' }).format(daysLeft, 'day');  // "in 176 days"
```

`Date` months start at 0 (`new Date(2027, 2, 31)` is 31 **March**). For heavy date work use a library such as `date-fns`. The new built-in `Temporal` API fixes most `Date` problems; check browser support before relying on it.

## 3.9 JSON

**What it is:** JSON (JavaScript Object Notation) is the text format APIs use to send data. It looks like a JS object, but keys must be in double quotes and there are no functions, `undefined`, or comments.

```js
const json = JSON.stringify({ id: 7, name: 'Amoxicillin' });   // object → text
// '{"id":7,"name":"Amoxicillin"}'
JSON.stringify(sale, null, 2);           // pretty-printed

const data = JSON.parse(json);            // text → object (throws SyntaxError on bad JSON)
```

**Structured data** in JS means objects, arrays and JSON. Dates become strings in JSON, so convert them back with `new Date(data.createdAt)`.
