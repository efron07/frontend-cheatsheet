# 21. Proxy, authentication, environment variables and assets

## 21.1 Proxy (formerly middleware)

**What it is:** `proxy.ts` in the project root (or `src/`) runs **before** a request reaches a route. It can redirect, rewrite, or set headers and cookies. In Next.js 16 the file and function were renamed from `middleware` to `proxy`, and it runs on the Node.js runtime.

**Use it for fast, simple checks** (is there a session cookie? which language?). Do the real permission checks next to the data (section 21.2), because proxy runs before the page and should stay light.

```ts
// src/proxy.ts
import { NextResponse, type NextRequest } from 'next/server';

export function proxy(request: NextRequest) {
  const session = request.cookies.get('session')?.value;
  const { pathname } = request.nextUrl;

  // Not logged in → send to login, remember where they were going
  if (pathname.startsWith('/dashboard') && !session) {
    const url = new URL('/login', request.url);
    url.searchParams.set('next', pathname);
    return NextResponse.redirect(url);
  }

  // Already logged in → skip the login page
  if (pathname === '/login' && session) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }

  return NextResponse.next();
}

// Route matcher: which paths run the proxy (skip static files for speed)
export const config = {
  matcher: ['/dashboard/:path*', '/login'],
};
```

## 21.2 Authentication

**What it is:** Verifying who the user is (login) and keeping them logged in across requests (a session). **Authorisation** is then checking what they may do.

**Options:**

| Approach | When |
| --- | --- |
| **Laravel API + Sanctum token in an HttpOnly cookie** | Laravel owns users (PharmaCare) — shown below |
| **Better Auth** | Next.js owns the database; email/password, OAuth, 2FA, organisations |
| **Auth.js (NextAuth)** | Very common in existing Next.js projects |
| **Clerk, Auth0, Kinde, Supabase Auth** | Hosted services: fastest setup, monthly cost |

**PharmaCare: log in against the Laravel API from a Server Action.**

```ts
// app/(auth)/login/actions.ts
'use server';
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';
import { z } from 'zod';

const LoginSchema = z.object({ email: z.email(), password: z.string().min(1) });

export async function login(_: unknown, formData: FormData) {
  const parsed = LoginSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { error: 'Enter a valid email and password.' };

  const res = await fetch(`${process.env.API_URL}/token`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Accept: 'application/json' },
    body: JSON.stringify({ ...parsed.data, device: 'pharmacare-web' }),
  });
  if (!res.ok) return { error: 'Invalid email or password.' };

  const { token } = await res.json();
  (await cookies()).set('session', token, {
    httpOnly: true,          // JavaScript cannot read it → safe from XSS theft
    secure: true,            // HTTPS only
    sameSite: 'lax',         // basic CSRF protection
    path: '/',
    maxAge: 60 * 60 * 12,    // 12 hours (one shift)
  });

  const next = String(formData.get('next') ?? '/dashboard');
  redirect(next.startsWith('/') ? next : '/dashboard');   // only internal redirects
}

export async function logout() {
  (await cookies()).delete('session');
  redirect('/login');
}
```

**A Data Access Layer (DAL)** — one place that verifies the session and returns the user. Every page, action and route handler calls it.

```ts
// lib/auth.ts
import 'server-only';
import { cache } from 'react';
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';

export const getCurrentUser = cache(async () => {
  const token = (await cookies()).get('session')?.value;
  if (!token) return null;
  const res = await fetch(`${process.env.API_URL}/me`, { headers: { Authorization: `Bearer ${token}`, Accept: 'application/json' } });
  return res.ok ? ((await res.json()) as { id: number; name: string; role: Role }) : null;
});

export async function requireUser() {
  const user = await getCurrentUser();
  if (!user) redirect('/login');
  return user;
}

export async function requireRole(roles: Role[]) {
  const user = await requireUser();
  if (!roles.includes(user.role)) throw new Error('Forbidden');
  return user;
}
```

```tsx
// app/(staff)/dashboard/reports/page.tsx
export default async function ReportsPage() {
  await requireRole(['admin', 'pharmacist']);      // checked on the server, every request
  /* ... */
}
```

**Handling sensitive data:** keep tokens and secrets on the server; pass only the fields the UI needs to Client Components (not the whole user record); never put secrets in `NEXT_PUBLIC_` variables.

## 21.3 Environment variables

```ini
# .env.local — never commit
API_URL=https://api.pharmacare.co.tz/api       # server only
API_TOKEN=secret-service-token                  # server only
DATABASE_URL=postgresql://user:pass@host/db     # server only
NEXT_PUBLIC_SITE_URL=https://pharmacare.co.tz   # NEXT_PUBLIC_ = bundled into browser JS
```

| File | Loaded |
| --- | --- |
| `.env` | Always (defaults; may be committed if it holds no secrets) |
| `.env.local` | Always, overrides `.env` (secrets; git-ignored) |
| `.env.development` / `.env.production` | Only in that mode |

- Read with `process.env.API_URL`.
- Only variables starting with **`NEXT_PUBLIC_`** reach the browser, and they are fixed into the JS **at build time**.
- Validate them at startup with Zod (or the `@t3-oss/env-nextjs` package) so a missing value fails the build, not a customer's checkout.

## 21.4 Images: `next/image`

**What it is:** A replacement for `<img>` that automatically resizes, converts to modern formats (WebP/AVIF), lazy-loads, and reserves space to prevent layout shift.

```tsx
import Image from 'next/image';
import logo from '@/public/logo.png';                 // static import: size known automatically

<Image src={logo} alt="PharmaCare" priority />          {/* priority: above-the-fold, load first */}

<Image
  src={medicine.imageUrl}
  alt={`${medicine.name} package`}
  width={400}
  height={300}
  sizes="(min-width: 1024px) 25vw, (min-width: 640px) 50vw, 100vw"   // which size to download
  className="aspect-[4/3] object-cover"
/>

{/* Unknown size: fill the parent (parent needs position + size) */}
<div className="relative aspect-square"><Image src={url} alt="" fill className="object-contain" /></div>
```

```ts
// next.config.ts — allow remote images from your API/CDN
images: { remotePatterns: [new URL('https://cdn.pharmacare.co.tz/**')] },
```

## 21.5 Fonts: `next/font`

**What it is:** Downloads Google or local fonts at build time and self-hosts them: no request to Google, no layout shift.

```tsx
import { Inter } from 'next/font/google';
import localFont from 'next/font/local';

const inter = Inter({ subsets: ['latin'], variable: '--font-sans', display: 'swap' });
const brand = localFont({ src: './fonts/Brand.woff2', variable: '--font-brand' });

<html className={`${inter.variable} ${brand.variable}`}>   {/* used by Tailwind's --font-sans */}
```

## 21.6 Scripts, analytics and third-party libraries

```tsx
import Script from 'next/script';

<Script src="https://widget.example.com/chat.js" strategy="lazyOnload" />   {/* load when idle */}
// strategies: beforeInteractive | afterInteractive (default) | lazyOnload | worker
```

- `@next/third-parties` provides optimised Google Analytics, Tag Manager, YouTube and Maps embeds.
- `@vercel/analytics` and `@vercel/speed-insights` track visits and Core Web Vitals on Vercel; the `useReportWebVitals` hook sends them anywhere.
- **Videos:** self-host with `<video>` (compressed MP4/WebM, `preload="none"`, a poster image) or lazy-load a YouTube embed.
- **Libraries that touch `window`** must run in Client Components; load heavy ones on demand: `const Chart = dynamic(() => import('./sales-chart'), { ssr: false, loading: () => <Skeleton /> })` (inside a Client Component).

## 21.7 Static assets and styles

- Files in `public/` are served from the site root: `public/favicon.ico` → `/favicon.ico`. Special files like `app/icon.png`, `app/apple-icon.png` and `app/opengraph-image.png` are picked up automatically.
- Styling options in Next.js: global CSS (imported in the root layout), Tailwind, CSS Modules, Sass. CSS-in-JS libraries need extra setup and only work in Client Components.

## 21.8 Internationalisation (i18n)

For English + Swahili: put the locale in the URL (`/en/medicines`, `/sw/medicines`) with an `app/[lang]/` segment, detect the preferred language in `proxy.ts`, and load translation dictionaries in Server Components. Libraries: **next-intl** (most popular), Lingui, i18next. Set `<html lang={lang}>` and `hreflang` alternates for SEO (section 23).

## 21.9 Markdown and MDX

Write content pages (health articles, FAQs) in Markdown, or **MDX** (Markdown that can embed React components) with `@next/mdx`. Pages become static and SEO-friendly.

## 21.10 Instrumentation and OpenTelemetry

`instrumentation.ts` in the root runs once when the server starts: set up monitoring (Sentry), OpenTelemetry tracing (`@vercel/otel`) or log shipping. `onRequestError` in the same file reports every server error to your monitoring tool.
