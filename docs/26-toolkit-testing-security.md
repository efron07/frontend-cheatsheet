# 26. Frontend toolkit 2: testing, security, accessibility and PWAs

## 26.1 Testing strategy

| Level | Tests | Tools | PharmaCare example |
| --- | --- | --- | --- |
| **Static** | Types and lint rules | TypeScript, ESLint | Wrong field name caught in the editor |
| **Unit** | Pure functions, reducers, schemas | Vitest | `cartReducer`, `formatTZS`, Zod `batchSchema` |
| **Component / integration** | Components with user interactions | Vitest + React Testing Library (+ MSW for API mocks) | Cart updates total when quantity changes |
| **End-to-end (E2E)** | Real browser, real app | Playwright (or Cypress) | Login → search → add to cart → complete sale |

Aim for many fast unit/component tests and a few E2E tests covering the money paths (login, sale, payment, stock receipt).

## 26.2 Vitest setup for Next.js

```bash
npm install -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/dom @testing-library/user-event @testing-library/jest-dom vite-tsconfig-paths
```

```ts
// vitest.config.mts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: { environment: 'jsdom', setupFiles: ['./vitest.setup.ts'], globals: true },
});
```

```ts
// vitest.setup.ts
import '@testing-library/jest-dom/vitest';
```

```ts
// lib/cart.test.ts — unit test
import { describe, it, expect } from 'vitest';
import { cartReducer } from './cart';

const amox = { id: 2, name: 'Amoxicillin', price: 500, stock: 2 } as Medicine;

describe('cartReducer', () => {
  it('adds a new medicine', () => {
    expect(cartReducer([], { type: 'add', medicine: amox })).toEqual([{ medicine: amox, qty: 1 }]);
  });

  it('never exceeds available stock', () => {
    let state = cartReducer([], { type: 'add', medicine: amox });
    state = cartReducer(state, { type: 'add', medicine: amox });
    state = cartReducer(state, { type: 'add', medicine: amox });   // third add ignored
    expect(state[0].qty).toBe(2);
  });
});
```

**Mock the API with MSW** (Mock Service Worker) so component tests do not hit the real server:

```ts
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

export const server = setupServer(
  http.get('*/api/medicines', () => HttpResponse.json({ data: [{ id: 1, name: 'Paracetamol', price: 300, stock: 120 }] })),
);
// in vitest.setup.ts: beforeAll(() => server.listen()); afterEach(() => server.resetHandlers()); afterAll(() => server.close());
```

Async Server Components are best covered by E2E tests; unit-test the data functions and Client Components.

**Jest** is the older alternative to Vitest with the same `describe/it/expect` style; you will see it in existing projects.

## 26.3 End-to-end tests with Playwright

```bash
npm init playwright@latest
npx playwright test             # run all
npx playwright test --ui        # interactive mode
npx playwright codegen localhost:3000   # record clicks into a test
```

```ts
// tests/e2e/sale.spec.ts
import { test, expect } from '@playwright/test';

test('cashier completes a cash sale', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('cashier@pharmacare.test');
  await page.getByLabel('Password').fill('password');
  await page.getByRole('button', { name: 'Log in' }).click();

  await page.goto('/dashboard/pos');
  await page.getByPlaceholder('Search or scan barcode').fill('Paracetamol');
  await page.getByRole('button', { name: 'Add' }).first().click();
  await expect(page.getByTestId('cart-total')).toHaveText('300 TZS');

  await page.getByRole('button', { name: 'Complete sale' }).click();
  await expect(page.getByText(/Receipt RCP-/)).toBeVisible();
});
```

Run E2E tests against `next build && next start` in CI, on Chromium, Firefox and WebKit (Safari).

## 26.4 Web security

| Threat | What it is | Defence in Next.js |
| --- | --- | --- |
| **XSS** (cross-site scripting) | Attacker's script runs in your page | React escapes `{}` output automatically; avoid `dangerouslySetInnerHTML` (sanitise with DOMPurify if unavoidable); add a CSP |
| **CSRF** (cross-site request forgery) | Another site triggers actions using the user's cookies | `SameSite` cookies; Server Actions only accept POST with origin checks |
| **Broken access control** | User reaches data/actions they should not | Auth + role check in every Server Action, Route Handler and data function (DAL) |
| **Injection** | Malicious input in SQL or commands | ORMs and parameterised queries; validate with Zod |
| **Secret leakage** | Keys shipped to the browser | Never prefix secrets `NEXT_PUBLIC_`; `import 'server-only'` |
| **Token theft** | Stolen session token | HttpOnly + Secure cookies, not localStorage; short expiry |
| **Clickjacking** | Your page loaded in a hidden iframe | `X-Frame-Options: DENY` / CSP `frame-ancestors 'none'` |
| **Vulnerable dependencies** | Known holes in packages | `npm audit`, Dependabot; keep Next.js patched (security releases) |
| **Open redirects** | `?next=https://evil.com` | Only redirect to paths starting with `/` |

**Content Security Policy (CSP)** — tells the browser which sources of scripts, styles and images are allowed. Set it in `next.config.ts` headers, or in `proxy.ts` with a per-request nonce for strict policies.

**CORS** (Cross-Origin Resource Sharing) — the browser rule that blocks a page on one domain from reading another domain's API unless that API allows it. Configure it on the API (Laravel `config/cors.php`), not in the frontend. Calls made from Next.js **server** code are not affected by CORS.

**HTTPS everywhere**, with HSTS (`Strict-Transport-Security`) once you are sure. Rate-limit login and sensitive endpoints. Log security events. Follow the **OWASP Top 10** as a checklist.

## 26.5 Accessibility (a11y)

**Standard:** WCAG 2.2 level AA. In practice:

- **Semantic HTML** first (section 10): buttons, links, headings, lists, tables, labels.
- **Keyboard:** every action works with Tab/Shift+Tab/Enter/Space/Escape; visible focus (`focus-visible:` styles); logical tab order; skip link to main content.
- **Screen readers:** meaningful `alt` text; labels on all inputs; `aria-live="polite"` region for toasts ("Sale completed"); `aria-describedby` linking an input to its error.
- **Colour:** contrast ≥ 4.5:1 for text; never colour-only meaning (expired batches: red **and** an "Expired" label).
- **Motion:** respect `prefers-reduced-motion`.
- **Forms:** clear errors next to the field, focus the first error on submit.
- **Components:** use accessible headless libraries (Radix, React Aria) for dialogs, menus, comboboxes.

**Test it:** keyboard-only walkthrough; the Lighthouse accessibility audit; the **axe DevTools** extension; screen readers (NVDA on Windows, VoiceOver on Mac/iPhone, TalkBack on Android); `eslint-plugin-jsx-a11y` (included in Next.js' ESLint config); `@axe-core/playwright` in E2E tests.

## 26.6 Progressive Web Apps (PWAs)

**What it is:** A PWA is a website that can be **installed** on a phone or desktop like an app, open full-screen, receive push notifications and work **offline**. Useful for a POS that must keep working when the internet drops.

```ts
// app/manifest.ts → /manifest.webmanifest
import type { MetadataRoute } from 'next';

export default function manifest(): MetadataRoute.Manifest {
  return {
    name: 'PharmaCare POS',
    short_name: 'PharmaCare',
    start_url: '/dashboard/pos',
    display: 'standalone',
    background_color: '#ffffff',
    theme_color: '#15803d',
    icons: [
      { src: '/icon-192.png', sizes: '192x192', type: 'image/png' },
      { src: '/icon-512.png', sizes: '512x512', type: 'image/png' },
    ],
  };
}
```

- A **service worker** (`public/sw.js`) caches the app shell and assets so the POS opens offline; **Serwist** is a popular library to generate one for Next.js.
- Queue sales made offline in **IndexedDB** and sync them when back online (Background Sync or on the `online` event).
- **Web Push** notifications (low stock alerts) need a service worker plus VAPID keys.

## 26.7 Other frontend topics on the roadmap

- **Web Components** — browser-native custom elements (`<pharma-badge>`), framework-independent; useful for widgets shared across sites.
- **GraphQL** — an alternative API style; clients choose exactly which fields to fetch.
- **Static site generators** — Astro, Eleventy for content-heavy sites.
- **Mobile and desktop** — React Native/Expo, Electron/Tauri (section 15.9).
- **Type checkers, monorepos, design systems** — grow into these as the team and codebase grow.
