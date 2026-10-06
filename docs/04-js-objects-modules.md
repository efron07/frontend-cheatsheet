# 4. Prototypes, classes, iterators and modules

## 4.1 Prototypes and prototypal inheritance

**What it is:** Every JS object has a hidden link to another object called its **prototype**. When you read a property the object does not have, JS looks it up on the prototype, then that prototype's prototype, and so on (the **prototype chain**). That is how arrays get `.map()`: it lives on `Array.prototype`, shared by every array.

```js
const baseProduct = {
  label() { return `${this.name} (${this.price} TZS)`; },
};

const paracetamol = Object.create(baseProduct);   // prototype = baseProduct
paracetamol.name = 'Paracetamol';
paracetamol.price = 300;
paracetamol.label();                               // found on the prototype

Object.getPrototypeOf(paracetamol) === baseProduct; // true
[].map === Array.prototype.map;                     // true
```

Classes (next) are a cleaner syntax on top of prototypes. Understanding prototypes explains why methods are shared and how `instanceof` works.

## 4.2 Classes

**What it is:** A class is a blueprint for creating objects that share structure and behaviour.

```js
class Product {
  #costPrice;                            // private field (# = only usable inside the class)
  static count = 0;                      // static: belongs to the class, not each object

  constructor(name, price, costPrice) {
    this.name = name;
    this.price = price;
    this.#costPrice = costPrice;
    Product.count++;
  }

  get margin() {                         // getter: read like a property
    return this.price - this.#costPrice;
  }

  set discountPercent(p) {               // setter: write like a property
    this.price = Math.round(this.price * (1 - p / 100));
  }

  label() {
    return `${this.name} — ${this.price} TZS`;
  }

  static fromApi(json) {                 // static factory method
    return new Product(json.name, json.price, json.cost_price);
  }
}

class Medicine extends Product {        // inheritance
  constructor(name, price, costPrice, requiresPrescription) {
    super(name, price, costPrice);       // call the parent constructor first
    this.requiresPrescription = requiresPrescription;
  }

  label() {                              // override
    return super.label() + (this.requiresPrescription ? ' [Rx]' : '');
  }
}

const amox = new Medicine('Amoxicillin', 500, 350, true);
amox.label();          // "Amoxicillin — 500 TZS [Rx]"
amox.margin;           // 150
amox instanceof Product;   // true
// amox.#costPrice;    // SyntaxError: private
```

In React apps you mostly write plain functions and objects, but classes appear in error types, SDK clients (a `PharmaApi` client) and libraries.

## 4.3 Symbols

**What it is:** A `Symbol` is a guaranteed-unique value, often used as an object key that cannot clash with other keys. Built-in "well-known" symbols like `Symbol.iterator` let you plug into language features.

```js
const internalId = Symbol('internalId');
const med = { name: 'Ibuprofen', [internalId]: 'x-91' };
Object.keys(med);    // ['name'] — symbol keys are hidden from normal listing
```

## 4.4 Iterators and generators

**What it is:** An **iterable** is anything `for...of` can loop over (arrays, strings, Map, Set). An **iterator** is the object that hands out values one at a time with `next()`. A **generator** (`function*`) is the easy way to write one: it pauses at each `yield`.

```js
// Generate receipt numbers on demand
function* receiptNumbers(prefix, start = 1) {
  let n = start;
  while (true) {
    yield `${prefix}-${String(n++).padStart(5, '0')}`;
  }
}

const gen = receiptNumbers('RCP');
gen.next().value;   // "RCP-00001"
gen.next().value;   // "RCP-00002"

// Make your own object iterable
const stockroom = {
  batches: [{ batchNo: 'A1' }, { batchNo: 'B2' }],
  *[Symbol.iterator]() {
    yield* this.batches;
  },
};
for (const b of stockroom) console.log(b.batchNo);
```

**Async generators** (`async function*` with `for await...of`) read data that arrives over time, such as pages of an API.

```js
async function* allMedicines() {
  let page = 1;
  while (true) {
    const res = await fetch(`/api/medicines?page=${page++}`);
    const { data, meta } = await res.json();
    yield* data;
    if (page > meta.last_page) return;
  }
}
for await (const m of allMedicines()) console.log(m.name);
```

## 4.5 Modules: ESM and CommonJS

**What it is:** A module is one file with its own scope that **exports** what others may use and **imports** what it needs. This is how every modern JS project is organised.

**ES Modules (ESM)** — the standard, used by browsers, React, Next.js:

```js
// lib/money.js
export const VAT = 18;                              // named export
export function formatTZS(amount) {
  return new Intl.NumberFormat('en-TZ').format(amount) + ' TZS';
}
export default function lineTotal(price, qty) {     // default export (one per file)
  return price * qty;
}

// app.js
import lineTotal, { formatTZS, VAT } from './lib/money.js';
import * as money from './lib/money.js';            // everything as one object
import { formatTZS as fmt } from './lib/money.js';  // rename

// Dynamic import: load only when needed (code splitting)
const { generateReceiptPdf } = await import('./lib/pdf.js');
```

**CommonJS (CJS)** — the older Node.js format. You will still see it in config files and older packages:

```js
// CommonJS
const express = require('express');
module.exports = { formatTZS };
```

|  | ESM | CommonJS |
| --- | --- | --- |
| Syntax | `import` / `export` | `require` / `module.exports` |
| Loading | Static, analysed before running (enables tree-shaking) | Dynamic, at runtime |
| Where | Browsers, modern Node, all modern frameworks | Older Node code, some config files |
| File hint | `.mjs`, or `"type": "module"` in `package.json` | `.cjs`, or no `type` field |

Rule: prefer **named exports** for utilities (easier to find, rename and tree-shake). Next.js requires **default exports** for pages and layouts.

## 4.6 Strict mode

**What it is:** A safer version of JS that turns silent mistakes into errors (e.g. assigning to an undeclared variable throws instead of creating a global). Enable with `'use strict';` at the top of a file. **ES modules and classes are always strict**, so in modern projects it is on by default.

## 4.7 Built-in objects worth knowing

`Object`, `Array`, `String`, `Number`, `Boolean`, `BigInt`, `Symbol`, `Math`, `Date`, `RegExp`, `JSON`, `Map`, `Set`, `WeakMap`, `WeakSet`, `Promise`, `Proxy`, `Reflect`, `Intl`, `Error` types, typed arrays, `globalThis` (the global object in any environment: `window` in browsers, `global` in Node).
