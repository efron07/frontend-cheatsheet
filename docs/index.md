# Frontend Cheat Sheet — JavaScript to Next.js

## How to use this sheet

This sheet takes you from zero JavaScript to production Next.js apps, in the order each layer builds on the last. It covers the roadmap.sh JavaScript, TypeScript, React, Next.js, Frontend, SEO, Frontend Performance and Tailwind CSS roadmaps.

**The learning path:**

1. **JavaScript** (sections 1–6) — the language itself. Everything else is built on it.
2. **TypeScript** (sections 7–9) — JavaScript with types, so mistakes show up in your editor instead of in front of customers.
3. **HTML, CSS and Tailwind** (sections 10–11) — structure and styling, which React components output.
4. **React** (sections 12–16) — building interfaces from components.
5. **Next.js** (sections 17–22) — React plus routing, server rendering, data fetching and deployment.
6. **SEO and performance** (sections 23–24) — being found on Google and loading fast.
7. **Toolkit** (sections 25–26) — Git, packages, testing, security and accessibility.
8. **Quick reference** (section 27) — commands and snippets.

Every topic follows the same pattern: **what it is** in plain words, **why it matters**, **how to do it**, then a **PharmaCare example**. Code versions targeted: modern JavaScript (ES2024+), TypeScript 5.x, React 19, Next.js 16 (App Router), Tailwind CSS v4.

## The example project: PharmaCare frontend

PharmaCare is the same pharmacy from the Laravel cheat sheet. Here we build its frontend in two parts:

- **Public site** (`pharmacare.co.tz`) — customers browse medicines, check availability and find the branch. Server-rendered for SEO.
- **Staff app** (`/dashboard`) — point of sale (POS), stock, batches, reports. Logged-in staff only, highly interactive.

The data comes from the Laravel API (or directly from a database in section 22). These are the shapes used in every example:

```ts
// types/pharmacy.ts
export type Role = 'admin' | 'pharmacist' | 'cashier';

export interface Category {
  id: number;
  name: string;            // "Antibiotics"
}

export interface Medicine {
  id: number;
  name: string;            // "Amoxicillin 500mg"
  slug: string;            // "amoxicillin-500mg"
  price: number;           // whole TZS, e.g. 500
  requiresPrescription: boolean;
  stock: number;           // total units across batches
  category: Category;
  imageUrl?: string;
}

export interface Batch {
  id: number;
  medicineId: number;
  batchNo: string;         // "AMX-2026-01"
  quantity: number;
  expiresAt: string;       // ISO date "2027-03-31"
}

export interface CartItem {
  medicine: Medicine;
  qty: number;
}

export type PaymentMethod = 'cash' | 'mpesa' | 'airtel' | 'card';

export interface Sale {
  id: number;
  receiptNo: string;
  items: { medicineId: number; qty: number; price: number }[];
  total: number;
  paymentMethod: PaymentMethod;
  createdAt: string;
}
```

Money is always whole TZS integers. Format it only when displaying: `new Intl.NumberFormat('en-TZ').format(1500)` gives "1,500".

## Sections

| Part | Pages |
| --- | --- |
| **JavaScript** | [1 Basics](01-js-basics.md) · [2 Functions](02-js-functions.md) · [3 Data](03-js-data.md) · [4 Objects & modules](04-js-objects-modules.md) · [5 Async](05-js-async.md) · [6 Browser](06-js-browser.md) |
| **TypeScript** | [7 Basics](07-ts-basics.md) · [8 Shaping data](08-ts-shaping-data.md) · [9 Generics & advanced](09-ts-generics.md) |
| **Styling** | [10 HTML & CSS](10-html-css.md) · [11 Tailwind CSS](11-tailwind.md) |
| **React** | [12 Fundamentals](12-react-fundamentals.md) · [13 State & events](13-react-state.md) · [14 Hooks](14-react-hooks.md) · [15 Architecture](15-react-architecture.md) · [16 Forms, UI, testing](16-react-forms-testing.md) |
| **Next.js** | [17 Routing](17-next-routing.md) · [18 Rendering](18-next-rendering.md) · [19 Data & caching](19-next-data.md) · [20 Mutations](20-next-mutations.md) · [21 Proxy & auth](21-next-proxy-auth.md) · [22 Deploy](22-next-deploy.md) |
| **Grow** | [23 SEO](23-seo.md) · [24 Performance](24-performance.md) · [25 Web, Git, tools](25-toolkit-web-git.md) · [26 Testing, security, a11y](26-toolkit-testing-security.md) |
| **Reference** | [27 Quick reference](27-quick-reference.md) |

Tip: press `/` or `S` on any page to search the whole sheet.
