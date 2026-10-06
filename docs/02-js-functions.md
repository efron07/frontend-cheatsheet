# 2. Control flow and functions

## 2.1 Conditional statements

**What it is:** Code that runs only when a condition is true.

```js
function stockLabel(stock, reorderLevel) {
  if (stock === 0) {
    return 'Out of stock';
  } else if (stock <= reorderLevel) {
    return 'Low stock';
  } else {
    return 'In stock';
  }
}

// Ternary: a one-line if/else that returns a value
const badge = stock > 0 ? 'green' : 'red';

// switch: compare one value against many cases (uses ===)
function paymentLabel(method) {
  switch (method) {
    case 'cash':   return 'Cash';
    case 'mpesa':  return 'M-Pesa';
    case 'airtel': return 'Airtel Money';
    default:       return 'Other';
  }
}
```

If a `case` has no `return` or `break`, execution "falls through" into the next case. Usually a bug.

**Lookup object instead of a long switch** (cleaner):

```js
const PAYMENT_LABELS = { cash: 'Cash', mpesa: 'M-Pesa', airtel: 'Airtel Money', card: 'Card' };
const label = PAYMENT_LABELS[method] ?? 'Other';
```

## 2.2 Loops

| Loop | Use for |
| --- | --- |
| `for` | Counting a known number of times |
| `while` | Repeat while a condition is true |
| `do...while` | Like `while`, but always runs at least once |
| `for...of` | Values of an iterable (array, string, Map, Set) — **most common** |
| `for...in` | Keys of an object (avoid on arrays) |
| Array methods | `forEach`, `map`, `filter` … (section 3) |

```js
const cart = [
  { name: 'Paracetamol', price: 300, qty: 2 },
  { name: 'Amoxicillin', price: 500, qty: 1 },
];

for (let i = 0; i < cart.length; i++) {
  console.log(i, cart[i].name);
}

for (const item of cart) {
  console.log(item.name, item.qty);
}

const medicine = { name: 'Ibuprofen', price: 800 };
for (const key in medicine) {
  console.log(key, medicine[key]);   // "name Ibuprofen", "price 800"
}

let attempts = 0;
while (attempts < 3) {
  attempts++;
}

do {
  // runs once even if the condition is false
} while (false);
```

**`break` and `continue`:** `break` exits the loop; `continue` skips to the next round.

```js
for (const item of cart) {
  if (item.qty === 0) continue;     // skip empty lines
  if (item.price < 0) break;        // stop on bad data
  console.log(item.name);
}
```

## 2.3 Errors and exception handling

**What it is:** When something goes wrong, JS **throws** an error object. If nothing **catches** it, the program stops. `try...catch...finally` lets you handle it.

```js
function sell(medicine, qty) {
  if (qty <= 0) throw new RangeError('Quantity must be at least 1');
  if (qty > medicine.stock) throw new Error(`Only ${medicine.stock} left`);
  return medicine.stock - qty;
}

try {
  sell({ name: 'Amoxicillin', stock: 5 }, 10);
} catch (error) {
  console.error(error.name, error.message);   // "Error Only 5 left"
} finally {
  console.log('Always runs: close the spinner, release the lock');
}
```

**Built-in error types:** `Error` (general), `TypeError` (wrong type, e.g. calling `undefined()`), `ReferenceError` (unknown variable), `RangeError` (number out of range), `SyntaxError` (invalid code or JSON).

**Your own error class:**

```js
class InsufficientStockError extends Error {
  constructor(medicineName, available) {
    super(`Only ${available} of ${medicineName} in stock`);
    this.name = 'InsufficientStockError';
    this.available = available;
  }
}

try {
  throw new InsufficientStockError('Amoxicillin', 3);
} catch (e) {
  if (e instanceof InsufficientStockError) showToast(e.message);
  else throw e;   // not ours: re-throw so it is not silently swallowed
}
```

## 2.4 Functions

**What it is:** A reusable block of code. You give it inputs (parameters), it does work, and returns an output.

```js
// 1. Function declaration (hoisted)
function lineTotal(price, qty) {
  return price * qty;
}

// 2. Function expression (stored in a variable, not hoisted)
const lineTotal2 = function (price, qty) {
  return price * qty;
};

// 3. Arrow function (short; no own `this`)
const lineTotal3 = (price, qty) => price * qty;         // implicit return
const toLabel = (m) => ({ label: m.name, value: m.id });  // return an object: wrap in ( )
```

**Parameters:**

```js
// Default parameters
function applyDiscount(total, percent = 0) {
  return Math.round(total * (1 - percent / 100));
}
applyDiscount(10000);      // 10000
applyDiscount(10000, 10);  // 9000

// Rest parameters: collect any number of arguments into an array
function sum(...amounts) {
  return amounts.reduce((a, b) => a + b, 0);
}
sum(300, 500, 1200);       // 2000

// Object parameter: named options, order does not matter
function createSale({ items, paymentMethod = 'cash', customerId = null }) { /* ... */ }
createSale({ items: cart, paymentMethod: 'mpesa' });
```

**The `arguments` object:** old-style functions (not arrows) get an array-like `arguments` with every value passed. Use rest parameters instead in modern code.

**Functions are values (first-class):** you can store them, pass them to other functions (callbacks), and return them.

```js
const byPrice = (a, b) => a.price - b.price;
medicines.sort(byPrice);                      // pass a function as an argument
```

**Recursion:** a function that calls itself. Needs a stopping condition.

```js
// Total stock of a category tree: Medicines > Antibiotics > Penicillins
function totalStock(category) {
  const own = category.medicines.reduce((s, m) => s + m.stock, 0);
  const children = category.children.reduce((s, c) => s + totalStock(c), 0);
  return own + children;
}
```

**IIFE** (Immediately Invoked Function Expression) runs once, right away. Older code used it to keep variables private; modules replace it today.

```js
(function () {
  const secret = 'private to this block';
})();
```

**Built-in global functions:** `parseInt`, `parseFloat`, `isNaN`, `isFinite`, `setTimeout`, `setInterval`, `structuredClone`, `encodeURIComponent`, `decodeURIComponent`.

## 2.5 The call stack

**What it is:** JS runs one thing at a time. Each function call is placed on a **stack**; when it returns, it is removed. An error message's "stack trace" is this list. Infinite recursion fills it and throws "Maximum call stack size exceeded".

## 2.6 Closures

**What it is:** A closure is a function that **remembers the variables around it** even after the outer function has finished. It is one of the most important ideas in JS and is used everywhere in React.

```js
function createCounter() {
  let count = 0;                    // private: nothing outside can touch it
  return {
    add: () => ++count,
    get: () => count,
  };
}

const receiptCounter = createCounter();
receiptCounter.add();   // 1
receiptCounter.add();   // 2
receiptCounter.get();   // 2
```

```js
// A function factory: make a price formatter per currency
function makeFormatter(currency) {
  const fmt = new Intl.NumberFormat('en-TZ', { style: 'currency', currency, maximumFractionDigits: 0 });
  return (amount) => fmt.format(amount);   // remembers `fmt`
}

const tzs = makeFormatter('TZS');
tzs(1500);   // "TSh 1,500"
```

## 2.7 The `this` keyword

**What it is:** `this` points to the object that is calling the function. Its value depends on **how** the function is called, not where it is written.

| Where | `this` is |
| --- | --- |
| A method: `cart.total()` | The object before the dot (`cart`) |
| A plain function call `fn()` | `undefined` in strict mode / modules |
| An arrow function | Inherited from the surrounding code (arrows have no own `this`) |
| A class constructor with `new` | The new object |
| A DOM event handler (`function`) | The element that received the event |
| Global scope (browser script) | `window` |

```js
const cart = {
  items: [{ price: 300, qty: 2 }],
  total() {
    return this.items.reduce((s, i) => s + i.price * i.qty, 0);  // arrow keeps `this` = cart
  },
};
cart.total();                 // 600

const loose = cart.total;
// loose();                   // TypeError: `this` is undefined — the method lost its object
```

**Explicit binding: `call`, `apply`, `bind`.** Set `this` yourself.

```js
function describe(prefix) {
  return `${prefix}: ${this.name}`;
}
const med = { name: 'Ibuprofen' };

describe.call(med, 'Medicine');      // call: arguments one by one
describe.apply(med, ['Medicine']);   // apply: arguments as an array
const bound = describe.bind(med);    // bind: returns a NEW function with `this` fixed
bound('Medicine');                   // "Medicine: Ibuprofen"
```

**Function borrowing** — use one object's method on another: `Array.prototype.slice.call(arguments)` turns `arguments` into a real array.

In React you will rarely use `this`, because components are functions. You still meet it in classes, libraries and old code.
