# 1. JavaScript basics

## 1.1 What is JavaScript?

**What it is:** JavaScript (JS) is the programming language of the web. HTML is the structure of a page, CSS is how it looks, and JavaScript is how it behaves: reacting to clicks, loading data, updating the page without a reload.

**Where it runs:**

- **In the browser** — every browser has a JS engine (Chrome and Edge use V8, Firefox uses SpiderMonkey, Safari uses JavaScriptCore).
- **On the server** — Node.js, Bun and Deno run JS outside the browser. Next.js uses Node.js on the server.

**A short history:** Brendan Eich created JavaScript in 10 days at Netscape in 1995. It is standardised as **ECMAScript** (ES). ES2015 ("ES6") was the big modern update: `let`, `const`, arrow functions, classes, modules, promises. Since then a new version ships every year (ES2016, ES2017 …). "Modern JS" means ES2015+.

## 1.2 How to run JavaScript

| Way | How | Good for |
| --- | --- | --- |
| Browser console | F12 → Console tab, type code, press Enter | Quick experiments |
| `<script>` in HTML | `<script src="app.js" defer></script>` | Plain web pages |
| Node.js | `node app.js` in a terminal | Scripts, servers, tooling |
| Online playgrounds | StackBlitz, CodeSandbox, JS Bin | Sharing examples |

```html
<!-- index.html -->
<script src="app.js" defer></script>   <!-- defer: run after the HTML is parsed -->
<script type="module" src="main.js"></script>  <!-- modules are deferred automatically -->
```

```js
// app.js
console.log('PharmaCare loaded');
console.table([{ name: 'Paracetamol', stock: 120 }, { name: 'Amoxicillin', stock: 8 }]);
console.error('Something failed');   // red in the console
```

## 1.3 Variables: `let`, `const`, `var`

**What it is:** A variable is a named box that holds a value.

| Keyword | Can reassign? | Scope | Use it? |
| --- | --- | --- | --- |
| `const` | No | Block `{ }` | **Default choice** |
| `let` | Yes | Block `{ }` | When the value must change |
| `var` | Yes | Function | Avoid (old style, confusing scope) |

```js
const pharmacyName = 'PharmaCare';      // never reassigned
let stock = 120;
stock = stock - 2;                      // OK: let can change

const cart = [];
cart.push('Paracetamol');               // OK: const stops REASSIGNING, not changing contents
// cart = [];                           // TypeError: Assignment to constant variable
```

**Naming rules:** start with a letter, `_` or `$`; no spaces; case-sensitive (`stock` and `Stock` differ); cannot be a reserved word (`class`, `return`). Convention: `camelCase` for variables and functions, `PascalCase` for classes and React components, `UPPER_SNAKE_CASE` for fixed constants like `MAX_CART_ITEMS`.

## 1.4 Scope and hoisting

**Scope** is where a variable can be seen.

- **Global scope** — declared outside everything; visible everywhere (avoid).
- **Function scope** — declared inside a function; visible only inside it.
- **Block scope** — `let`/`const` inside `{ }` (an `if`, a loop) are visible only inside that block.
- **Lexical scoping** — an inner function can see the variables of the functions around it, based on where it is *written*.

```js
const vat = 18;                     // global

function priceWithVat(price) {
  const multiplier = 1 + vat / 100; // function scope; can see vat (lexical scope)
  if (price > 0) {
    const result = price * multiplier;  // block scope
    return result;
  }
  // console.log(result);           // ReferenceError: not visible here
}
```

**Hoisting:** before running, JS moves declarations to the top of their scope.

```js
console.log(a);   // undefined   — var is hoisted and set to undefined
var a = 1;

console.log(b);   // ReferenceError — let/const are hoisted but untouchable until declared
let b = 2;        // (the gap before this line is called the "temporal dead zone")

greet();          // works — function declarations are fully hoisted
function greet() { console.log('Karibu'); }
```

## 1.5 Data types

JavaScript has **7 primitive types** (simple values, copied by value) and **objects** (copied by reference).

| Type | Example | Notes |
| --- | --- | --- |
| `string` | `'Paracetamol'`, `"500mg"`, `` `Total: ${total}` `` | Text |
| `number` | `500`, `12.5`, `NaN`, `Infinity` | One type for integers and decimals |
| `bigint` | `9007199254740993n` | Integers bigger than `Number.MAX_SAFE_INTEGER` |
| `boolean` | `true`, `false` |  |
| `undefined` | `let x;` | "No value assigned yet" |
| `null` | `let supplier = null;` | "Intentionally empty" |
| `symbol` | `Symbol('id')` | A unique value, used as hidden object keys |
| `object` | `{ name: 'Ibuprofen' }`, `[1, 2]`, `new Date()`, functions | Everything else |

```js
typeof 'abc'        // "string"
typeof 500          // "number"
typeof 10n          // "bigint"
typeof true         // "boolean"
typeof undefined    // "undefined"
typeof Symbol()     // "symbol"
typeof {}           // "object"
typeof []           // "object"   — use Array.isArray([]) to check arrays
typeof null         // "object"   — a famous old bug; check with  x === null
typeof function(){} // "function"
```

**Floats warning (money!):** `0.1 + 0.2` is `0.30000000000000004`. Never store money as decimals. Keep whole TZS (or cents) as integers.

**Primitive vs reference:**

```js
let a = 5; let b = a; b = 10;        // a is still 5 (copied)

const m1 = { name: 'Ibuprofen' };
const m2 = m1;                        // same object, two names
m2.name = 'Changed';
console.log(m1.name);                 // "Changed" — both point to one object
```

**Typed arrays** (`Uint8Array`, `Float32Array` …) hold raw binary numbers. You meet them when handling files, images or barcodes as bytes.

## 1.6 Type conversion and coercion

- **Explicit conversion (type casting):** you convert on purpose.
- **Implicit coercion:** JS converts automatically, sometimes surprisingly.

```js
// Explicit
Number('500')        // 500
Number('500abc')     // NaN
parseInt('500mg')    // 500     (reads digits until it hits a letter)
parseFloat('12.5')   // 12.5
String(500)          // "500"
Boolean('')          // false
(1500).toString()    // "1500"

// Implicit (coercion)
'5' + 2     // "52"   — + with a string joins text
'5' - 2     // 3      — other maths operators convert to number
'5' * '2'   // 10
true + 1    // 2
```

**Truthy and falsy:** in an `if`, every value becomes true or false. The **falsy** values are: `false`, `0`, `-0`, `0n`, `''`, `null`, `undefined`, `NaN`. Everything else is truthy, including `'0'`, `[]` and `{}`.

```js
const stock = 0;
if (!stock) console.log('Out of stock');   // runs: 0 is falsy
```

## 1.7 Operators

| Kind | Operators | Example |
| --- | --- | --- |
| Arithmetic | `+ - * / % **` | `qty * price`, `15 % 4` → 3, `2 ** 3` → 8 |
| Increment | `++ --` | `count++` |
| Assignment | `= += -= *= /= ??= \|\|= &&=` | `total += price` |
| Comparison | `=== !== > < >= <=` | `stock <= reorderLevel` |
| Logical | `&& \|\| !` | `isLoggedIn && isPharmacist` |
| Nullish | `??` | `discount ?? 0` |
| Optional chaining | `?.` | `sale.customer?.phone` |
| Ternary | `cond ? a : b` | `stock > 0 ? 'In stock' : 'Out'` |
| String | `+`, template literals | `` `${qty} × ${name}` `` |
| Unary | `typeof`, `delete`, `!`, `-`, `+` | `+'500'` → 500 |
| Bitwise | `& \| ^ ~ << >> >>>` | Rare: flags and low-level maths |
| Comma | `a, b` | Evaluates both, returns the last (rare) |

**`||` vs `??`:** `||` replaces any falsy value; `??` replaces only `null`/`undefined`. For a stock of `0`, that difference matters:

```js
const stock = 0;
stock || 'unknown'   // "unknown"  — wrong, 0 is real data
stock ?? 'unknown'   // 0          — correct
```

**Short-circuiting:** `&&` stops at the first falsy value, `||` at the first truthy one. Often used for defaults and conditional calls: `user && logActivity(user)`.

## 1.8 Equality

| Check | Name | Converts types? | Use |
| --- | --- | --- | --- |
| `==` | Loose equality (IsLooselyEqual) | Yes | Avoid |
| `===` | Strict equality (IsStrictlyEqual) | No | **Always** |
| `Object.is(a, b)` | SameValue | No; also `NaN` equals `NaN`, `+0` ≠ `-0` | Edge cases |

```js
0 == ''            // true   (surprise!)
0 === ''           // false
null == undefined  // true
null === undefined // false
NaN === NaN        // false  — NaN never equals itself
Number.isNaN(NaN)  // true   — the right way to check
Object.is(NaN, NaN) // true
```

**SameValueZero** is the comparison used by `includes`, `Map` and `Set`: like `===` but `NaN` equals `NaN`. So `[NaN].includes(NaN)` is `true` while `[NaN].indexOf(NaN)` is `-1`.

Objects are compared by reference, not by contents: `{ id: 1 } === { id: 1 }` is `false`.
