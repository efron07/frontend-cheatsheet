# 22. Data layer, configuration, structure and deployment

## 22.1 Two ways to get data: API or database

|  | Call the Laravel API | Query a database directly |
| --- | --- | --- |
| Who owns the business logic | Laravel (validation, stock rules, M-Pesa) | Next.js |
| Next.js role | Frontend + BFF (backend-for-frontend) | Full-stack app |
| Tools | `fetch` + Zod (sections 5, 19) | Prisma or Drizzle ORM |
| Good when | A backend already exists or is shared with a mobile app | New product, one codebase |

**Drizzle ORM** (SQL-like, lightweight) — example schema and query:

```ts
// lib/db/schema.ts
import { pgTable, serial, text, integer, boolean, date } from 'drizzle-orm/pg-core';

export const medicines = pgTable('medicines', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  price: integer('price').notNull(),                  // whole TZS
  requiresPrescription: boolean('requires_prescription').default(false).notNull(),
});

export const batches = pgTable('batches', {
  id: serial('id').primaryKey(),
  medicineId: integer('medicine_id').references(() => medicines.id).notNull(),
  batchNo: text('batch_no').notNull().unique(),
  quantity: integer('quantity').notNull(),
  expiresAt: date('expires_at').notNull(),
});
```

```ts
// lib/db/index.ts
import 'server-only';
import { drizzle } from 'drizzle-orm/node-postgres';
export const db = drizzle(process.env.DATABASE_URL!);

// usage in a Server Component or action
import { eq, ilike } from 'drizzle-orm';
const rows = await db.select().from(medicines).where(ilike(medicines.name, `%${search}%`)).limit(20);
```

```bash
npx drizzle-kit generate   # create SQL migrations from the schema
npx drizzle-kit migrate    # apply them
```

**Prisma** is the other popular choice (schema file + generated client: `db.medicine.findMany(...)`, used in earlier examples). Both are fully typed. Hosted Postgres options: Neon, Supabase, Vercel Postgres, Railway, or your own server.

## 22.2 `next.config.ts`

```ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  cacheComponents: true,                    // Next.js 16 caching model (section 19)
  reactCompiler: true,                      // automatic memoisation (section 16)
  output: 'standalone',                     // minimal server bundle for Docker / VPS
  images: { remotePatterns: [new URL('https://cdn.pharmacare.co.tz/**')] },
  experimental: { optimizePackageImports: ['lucide-react', 'date-fns'] },   // smaller bundles
  serverExternalPackages: ['@react-pdf/renderer'],                          // keep heavy libs out of the bundle
  async redirects() {
    return [{ source: '/shop/:slug', destination: '/medicines/:slug', permanent: true }];
  },
  async headers() {
    return [{
      source: '/(.*)',
      headers: [
        { key: 'X-Content-Type-Options', value: 'nosniff' },
        { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
        { key: 'X-Frame-Options', value: 'DENY' },
      ],
    }];
  },
};

export default nextConfig;
```

## 22.3 The PharmaCare project structure

```
pharmacare-web/
├── src/
│   ├── app/
│   │   ├── layout.tsx                 # html, fonts, <Providers>
│   │   ├── globals.css                # @import 'tailwindcss' + @theme
│   │   ├── sitemap.ts, robots.ts      # SEO (section 23)
│   │   ├── (public)/                  # customer site, server-rendered
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx               # home
│   │   │   ├── medicines/page.tsx     # catalogue + search
│   │   │   ├── medicines/[slug]/page.tsx
│   │   │   └── branches/page.tsx
│   │   ├── (auth)/login/{page.tsx,actions.ts}
│   │   ├── (staff)/dashboard/         # requires login
│   │   │   ├── layout.tsx             # sidebar; requireUser()
│   │   │   ├── page.tsx               # stats, streamed
│   │   │   ├── pos/page.tsx           # client-heavy POS
│   │   │   ├── stock/{page.tsx,actions.ts,_components/}
│   │   │   └── reports/page.tsx
│   │   └── api/
│   │       ├── mpesa/callback/route.ts
│   │       └── webhooks/stock/route.ts
│   ├── components/
│   │   ├── ui/                        # shadcn/ui: button, dialog, input, table
│   │   └── medicine-card.tsx, stock-badge.tsx, ...
│   ├── lib/
│   │   ├── api.ts                     # fetch wrapper for Laravel
│   │   ├── auth.ts                    # getCurrentUser, requireRole (DAL)
│   │   ├── data/                      # cached data functions ('use cache')
│   │   ├── env.ts                     # validated env variables
│   │   └── utils.ts                   # cn(), formatTZS()
│   ├── schemas/                       # Zod schemas shared by forms and actions
│   ├── stores/cart-store.ts           # Zustand
│   ├── hooks/                         # useDebounce, useOnline
│   ├── types/pharmacy.ts
│   └── proxy.ts
├── public/                            # logo, images
├── tests/e2e/                         # Playwright
├── next.config.ts
├── tsconfig.json
└── .env.local
```

**Structuring principles:** group by feature/route, keep server-only code in `lib/` behind `import 'server-only'`, keep `'use client'` files small, share Zod schemas between client and server.

## 22.4 Preparing for production

- [ ] `npm run build` passes with no type or lint errors.
- [ ] Environment variables set on the server; secrets not prefixed `NEXT_PUBLIC_`.
- [ ] Every Server Action and Route Handler checks auth and validates input.
- [ ] `error.tsx`, `not-found.tsx` and `loading.tsx` exist for main sections.
- [ ] Metadata, sitemap and robots configured (section 23).
- [ ] Images use `next/image`, fonts use `next/font`.
- [ ] Lighthouse score checked on a mobile profile (section 24).
- [ ] Error monitoring (Sentry) and analytics connected.
- [ ] Security headers set.

## 22.5 Deployment options

| Option | How | Notes |
| --- | --- | --- |
| **Vercel** | Connect the GitHub repo; every push deploys, every PR gets a preview URL | Made by the Next.js team; zero config; usage-based pricing |
| **Node.js server / VPS** | `npm run build && npm start` behind Nginx, kept alive with PM2 or systemd | Full control; you handle scaling and TLS |
| **Docker** | `output: 'standalone'` + a small Node image | Any cloud, Coolify, your own servers |
| **Static export** | `output: 'export'` → plain HTML/CSS/JS in `out/` | Any static host; no server features (no actions, no dynamic SSR) |
| **Other platforms** | Netlify, Cloudflare, AWS (Amplify, SST/OpenNext), Railway, Render | Use **adapters** that translate Next.js output to the platform |

**Dockerfile (standalone):**

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:22-alpine AS run
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

**Self-hosting notes:** put a CDN or Nginx in front for static files and TLS; when running several instances, share the cache with a custom cache handler (e.g. Redis); watch **memory usage** (a Node process that keeps growing usually means a leak or an over-large in-memory cache). A **custom server** (your own Express wrapper) is possible but disables some optimisations; avoid it unless you really need it.

## 22.6 Package bundling

- Analyse bundle size: `npx next experimental-analyze` or `@next/bundle-analyzer`.
- Import only what you use (`import { format } from 'date-fns'`), prefer lighter libraries, and load heavy client libraries with `dynamic()`.
- Keep big server-only packages out of client bundles by importing them only in Server Components or actions.
