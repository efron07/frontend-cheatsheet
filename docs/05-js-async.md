# 5. Asynchronous JavaScript

## 5.1 Why async exists

**What it is:** JS runs on **one thread**: it can do only one thing at a time. Some work takes a long time — loading medicines from the API, waiting for an M-Pesa confirmation, a timer. If JS waited for each one, the whole page would freeze. **Asynchronous** code starts the slow work, carries on with other things, and handles the result when it is ready.

## 5.2 The event loop

**What it is:** The mechanism that decides what runs next.

1. **Call stack** — the code running right now.
2. **Web APIs** (browser) or the Node runtime — do slow work in the background: timers, network requests, file reads.
3. **Microtask queue** — promise callbacks (`.then`, code after `await`). Highest priority.
4. **Task queue** (macrotasks) — `setTimeout`, `setInterval`, DOM events.
5. **The loop:** when the call stack is empty, run **all** microtasks, then the browser may repaint the screen, then take **one** task, and repeat.

```js
console.log('1 start');
setTimeout(() => console.log('4 timeout (task)'), 0);
Promise.resolve().then(() => console.log('3 promise (microtask)'));
console.log('2 end');
// Output: 1 start, 2 end, 3 promise (microtask), 4 timeout (task)
```

Lesson: `setTimeout(fn, 0)` does not mean "now"; it means "after the current code and all pending promises".

## 5.3 Timers: `setTimeout` and `setInterval`

```js
const id = setTimeout(() => showToast('Session expires in 1 minute'), 14 * 60 * 1000);
clearTimeout(id);                              // cancel

const poll = setInterval(checkPaymentStatus, 5000);   // every 5 seconds
clearInterval(poll);                           // ALWAYS stop intervals you start
```

## 5.4 Callbacks and callback hell

**What it is:** A callback is a function you pass to be called later. It was the original async pattern.

```js
// Callback hell: each step nested inside the last — hard to read and handle errors
getCustomer(phone, (customer) => {
  createSale(customer, cart, (sale) => {
    requestMpesa(sale, (payment) => {
      printReceipt(sale, payment, () => { /* ... */ });
    });
  });
});
```

Promises and `async/await` were created to fix this.

## 5.5 Promises

**What it is:** A Promise is an object representing a value that will arrive later. It is **pending**, then either **fulfilled** (success, with a value) or **rejected** (failure, with an error).

```js
// Creating a promise (you rarely need to; most APIs return one)
function wait(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function checkStock(id) {
  return new Promise((resolve, reject) => {
    const m = inventory.find((x) => x.id === id);
    m ? resolve(m.stock) : reject(new Error('Medicine not found'));
  });
}

// Using a promise
checkStock(2)
  .then((stock) => console.log('Stock:', stock))
  .catch((err) => console.error(err.message))
  .finally(() => setLoading(false));
```

**Combining promises:**

| Method | Waits for | Fails when | PharmaCare use |
| --- | --- | --- | --- |
| `Promise.all([...])` | All to succeed | Any one fails | Load medicines + categories + branches together |
| `Promise.allSettled([...])` | All to finish | Never; reports each result | Send SMS to 50 customers, see which failed |
| `Promise.race([...])` | The first to finish | First finisher fails | Request vs a timeout |
| `Promise.any([...])` | The first success | All fail | Try two mirror servers |

```js
const [medicines, categories] = await Promise.all([
  fetch('/api/medicines').then((r) => r.json()),
  fetch('/api/categories').then((r) => r.json()),
]);
```

## 5.6 `async` / `await`

**What it is:** A cleaner way to write promise code that reads top to bottom like normal code. An `async` function always returns a promise; `await` pauses that function (not the whole page) until the promise settles.

```js
async function completeSale(cart, phone) {
  try {
    const sale = await createSale(cart);            // wait for step 1
    const payment = await requestMpesa(sale, phone); // then step 2
    await printReceipt(sale, payment);
    return sale;
  } catch (error) {
    showToast(`Sale failed: ${error.message}`);
    throw error;
  }
}
```

**Sequential vs parallel** — a very common performance mistake:

```js
// Slow: second request waits for the first (sequential) — only do this if B needs A's result
const meds = await getMedicines();
const cats = await getCategories();

// Fast: both start immediately (parallel)
const [meds2, cats2] = await Promise.all([getMedicines(), getCategories()]);
```

**Top-level `await`** works directly in ES modules (no wrapper function needed).

## 5.7 Working with APIs: `fetch`

**What it is:** `fetch` sends HTTP requests from JS. It is built into browsers, Node.js and Next.js.

```js
const API = 'https://api.pharmacare.co.tz/api';

// GET
async function getMedicines(search = '') {
  const res = await fetch(`${API}/medicines?search=${encodeURIComponent(search)}`, {
    headers: { Accept: 'application/json' },
  });
  if (!res.ok) {                                   // fetch does NOT throw on 404/500!
    throw new Error(`Request failed: ${res.status}`);
  }
  return res.json();
}

// POST with JSON and a token
async function createSale(cart, token) {
  const res = await fetch(`${API}/sales`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Accept: 'application/json',
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify({
      payment_method: 'cash',
      items: cart.map((i) => ({ medicine_id: i.medicine.id, qty: i.qty })),
    }),
  });

  const data = await res.json();
  if (res.status === 422) throw new ValidationError(data.errors);   // Laravel validation errors
  if (!res.ok) throw new Error(data.message ?? 'Sale failed');
  return data;
}

// Upload a file (prescription scan): let the browser set Content-Type
const form = new FormData();
form.append('scan', fileInput.files[0]);
await fetch(`${API}/sales/15/prescription`, { method: 'POST', body: form, headers: { Authorization: `Bearer ${token}` } });
```

**Cancel a request with `AbortController`** — e.g. the user types a new search before the old one returns:

```js
let controller;
async function search(term) {
  controller?.abort();                         // cancel the previous request
  controller = new AbortController();
  try {
    const res = await fetch(`/api/medicines?search=${term}`, { signal: controller.signal });
    return await res.json();
  } catch (e) {
    if (e.name === 'AbortError') return;       // expected, ignore
    throw e;
  }
}

// Built-in timeout
await fetch(url, { signal: AbortSignal.timeout(8000) });   // give up after 8 seconds
```

**`XMLHttpRequest`** (XHR) is the older API that `fetch` replaced. You only meet it in old code; it is the base of the term "AJAX". **Axios** is a popular library that wraps requests (auto JSON, throws on errors, interceptors); `fetch` is enough for most Next.js apps.

## 5.8 A reusable API client

```js
// lib/api.js
export class ApiError extends Error {
  constructor(status, body) {
    super(body?.message ?? `HTTP ${status}`);
    this.status = status;
    this.errors = body?.errors ?? {};
  }
}

export async function api(path, { method = 'GET', body, token } = {}) {
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}${path}`, {
    method,
    headers: {
      Accept: 'application/json',
      ...(body && { 'Content-Type': 'application/json' }),
      ...(token && { Authorization: `Bearer ${token}` }),
    },
    body: body ? JSON.stringify(body) : undefined,
  });
  const data = res.status === 204 ? null : await res.json();
  if (!res.ok) throw new ApiError(res.status, data);
  return data;
}

// usage
const medicines = await api('/medicines?search=amox');
await api('/sales', { method: 'POST', body: { items }, token });
```
