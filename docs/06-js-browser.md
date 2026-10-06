# 6. JavaScript in the browser

## 6.1 The DOM

**What it is:** The DOM (Document Object Model) is the browser's live tree of objects built from your HTML. JS reads and changes this tree, and the browser redraws the page. React does this for you, but knowing the DOM explains what React is doing underneath.

```html
<input id="search" placeholder="Search medicine" />
<ul id="results"></ul>
<p class="total">Total: <span id="total">0</span> TZS</p>
```

```js
// Select
const input = document.getElementById('search');
const list = document.querySelector('#results');          // first match (CSS selector)
const prices = document.querySelectorAll('.price');       // all matches (NodeList)

// Read and change
document.querySelector('#total').textContent = '1,500';   // safe: plain text
list.innerHTML = '';                                       // clear (innerHTML parses HTML — XSS risk with user input)
input.value;                                               // current input text

// Create elements
function renderResults(medicines) {
  list.replaceChildren(
    ...medicines.map((m) => {
      const li = document.createElement('li');
      li.textContent = `${m.name} — ${m.price} TZS`;
      li.dataset.id = m.id;                                // <li data-id="7">
      if (m.stock === 0) li.classList.add('out-of-stock');
      return li;
    })
  );
}

// Classes, attributes, styles
el.classList.add('active'); el.classList.remove('active'); el.classList.toggle('open');
el.setAttribute('aria-expanded', 'true');
el.style.display = 'none';
el.remove();
```

## 6.2 Events

**What it is:** Events are things that happen — a click, a key press, a form submit. You attach a **listener** function to react.

```js
input.addEventListener('input', (event) => {
  const term = event.target.value;
  searchMedicines(term).then(renderResults);
});

// Event delegation: ONE listener on the parent handles clicks on any child,
// even children added later
list.addEventListener('click', (event) => {
  const li = event.target.closest('li');
  if (!li) return;
  addToCart(Number(li.dataset.id));
});

// Keyboard shortcut: F2 focuses search at the POS
document.addEventListener('keydown', (e) => {
  if (e.key === 'F2') { e.preventDefault(); input.focus(); }
});

// Remove a listener (needs the same function reference)
const onResize = () => { /* ... */ };
window.addEventListener('resize', onResize);
window.removeEventListener('resize', onResize);
```

**Bubbling:** an event fires on the target element, then "bubbles" up through each parent. `event.stopPropagation()` stops it; `event.preventDefault()` stops the browser's default action (following a link, submitting a form).

**Debounce** — wait until the user stops typing before searching:

```js
function debounce(fn, delay = 300) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);   // closure remembers `timer`
  };
}
input.addEventListener('input', debounce((e) => search(e.target.value), 300));
```

## 6.3 Forms

```html
<form id="stock-form">
  <input name="batchNo" required />
  <input name="quantity" type="number" min="1" required />
  <input name="expiresAt" type="date" required />
  <button>Receive stock</button>
</form>
```

```js
document.querySelector('#stock-form').addEventListener('submit', async (e) => {
  e.preventDefault();                                    // stop the page reload
  const form = e.currentTarget;
  const data = Object.fromEntries(new FormData(form));   // { batchNo, quantity, expiresAt } (all strings)
  data.quantity = Number(data.quantity);

  if (!form.checkValidity()) return form.reportValidity();
  await api('/batches', { method: 'POST', body: data, token });
  form.reset();
});
```

## 6.4 Browser storage

| Storage | Size | Lifetime | Sent to server? | Use for |
| --- | --- | --- | --- | --- |
| `localStorage` | \~5 MB | Until cleared | No | UI preferences (theme, last branch) |
| `sessionStorage` | \~5 MB | Until the tab closes | No | A draft cart in one tab |
| Cookies | \~4 KB | Set by expiry | **Yes, every request** | Login sessions (`HttpOnly`, set by the server) |
| IndexedDB | Large | Until cleared | No | Offline POS data, queued sales |

```js
localStorage.setItem('pharmacare:theme', 'dark');
localStorage.getItem('pharmacare:theme');
localStorage.setItem('cart', JSON.stringify(cart));          // only strings: stringify objects
const cart = JSON.parse(localStorage.getItem('cart') ?? '[]');
localStorage.removeItem('cart');
```

Rule: never store tokens or personal data in `localStorage`. Any script on the page can read it, so one XSS bug leaks it. Use `HttpOnly` cookies for sessions (section 21).

## 6.5 Other useful Web APIs

| API | What it does | PharmaCare example |
| --- | --- | --- |
| `navigator.clipboard` | Copy and paste | Copy receipt number |
| `navigator.onLine` + `online`/`offline` events | Network status | Show "Offline — sales will sync later" |
| `IntersectionObserver` | Detect when an element scrolls into view | Infinite scroll of medicines, lazy images |
| `ResizeObserver` | Detect element size changes | Responsive charts |
| `window.print()` | Open the print dialog | Print a receipt |
| `Notification` | Desktop notifications | "Low stock: Amoxicillin" |
| `navigator.geolocation` | User location (with permission) | Nearest branch |
| `URL`, `URLSearchParams` | Build and read URLs | `new URLSearchParams(location.search).get('q')` |
| `crypto.randomUUID()` | Random unique id | Idempotency key for a sale |
| `BroadcastChannel` | Message other tabs | Log out every open tab |
| Media/Camera (`getUserMedia`) | Camera access | Barcode scanning |

## 6.6 Memory management and garbage collection

**What it is:** JS frees memory automatically. The **garbage collector** removes objects that nothing can reach any more ("mark and sweep"). The **memory lifecycle** is: allocate (create a value) → use → release (when unreachable).

**Common memory leaks:**

- Event listeners or `setInterval` never removed (very common in React effects — always clean up).
- Growing global arrays or caches that are never trimmed.
- Detached DOM nodes still referenced from JS.
- Closures holding on to large objects longer than needed.

## 6.7 Debugging with browser DevTools

Open with **F12** (or Ctrl+Shift+I).

| Panel | Use it to |
| --- | --- |
| **Console** | See logs and errors, run code. `$0` is the element selected in Elements |
| **Elements** | Inspect and live-edit HTML and CSS |
| **Sources** | Set breakpoints, step through code line by line. Or write `debugger;` in your code |
| **Network** | See every request, status, timing, payload. Throttle to "Slow 4G" to test slow networks |
| **Application** | Inspect localStorage, cookies, IndexedDB, service workers |
| **Performance** | Record a session: find slow functions and long tasks (debugging performance) |
| **Memory** | Heap snapshots: compare two snapshots to find leaks (debugging memory leaks) |
| **Lighthouse** | Audit performance, accessibility, SEO (section 24) |

```js
console.log({ cart, total });     // object shorthand shows variable names
console.table(medicines);
console.time('render'); /* ... */ console.timeEnd('render');
console.trace();                  // print the call stack
```

Install the **React Developer Tools** browser extension once you reach React: it adds Components and Profiler panels.
