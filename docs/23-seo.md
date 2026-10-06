# 23. SEO

## 23.1 How search engines work

**What it is:** SEO (search engine optimisation) is making your pages easy for search engines to find, understand and rank, so customers searching "pharmacy Dodoma" or "amoxicillin price Tanzania" land on PharmaCare.

1. **Crawling** — bots (Googlebot) follow links and sitemaps to discover pages.
2. **Rendering** — they load the page; server-rendered HTML is understood immediately, JS-only content may be delayed or missed.
3. **Indexing** — the page's content is stored and understood (topic, language, entities).
4. **Ranking** — for each search, pages are ordered by relevance, quality, trust, page experience and links.

AI search (Google AI Overviews, ChatGPT, Perplexity) relies on the same foundations: clear, well-structured, trustworthy content that can be crawled.

**Health content is "Your Money or Your Life" (YMYL):** search engines hold medical pages to a higher standard. Show who wrote or reviewed content (a named pharmacist), cite sources, keep information accurate and dated.

## 23.2 Keyword research and content

- Find what people actually search: Google Search Console queries, Google's autocomplete and "People also ask", Keyword Planner, Ahrefs/Semrush. Include Swahili queries ("duka la dawa Dodoma").
- Map **one main topic per page** (search intent: informational "what is amoxicillin used for", commercial "amoxicillin price", local "pharmacy near me").
- Write for people first: answer the question early, use clear headings, original information (real prices, real stock, branch hours), no keyword stuffing.
- Keep content fresh: update prices, availability and articles.

## 23.3 On-page SEO

| Element | Best practice | PharmaCare example |
| --- | --- | --- |
| Title tag | Unique, \~50–60 characters, main keyword first | "Amoxicillin 500mg — Price & Availability |
| Meta description | \~150–160 characters; a reason to click (not a ranking factor, but affects clicks) | "In stock at 3 Dodoma branches. 500 TZS per capsule..." |
| URL | Short, readable, hyphens, lowercase | `/medicines/amoxicillin-500mg` |
| Headings | One `<h1>`, logical `h2`/`h3` | h1: medicine name; h2: Uses, Dosage, Availability |
| Images | Descriptive `alt`, compressed, modern format | `alt="Amoxicillin 500mg capsules, pack of 21"` |
| Internal links | Link related pages with descriptive text | Category → medicines; articles → products |
| Content | Original, complete, helpful | Uses, warnings, prescription requirement, branches |

## 23.4 Metadata in Next.js

```tsx
// app/layout.tsx — site-wide defaults
export const metadata: Metadata = {
  metadataBase: new URL('https://pharmacare.co.tz'),     // makes relative URLs absolute
  title: { default: 'PharmaCare — Pharmacy in Dodoma', template: '%s | PharmaCare' },
  description: 'Genuine medicines, fair prices, three branches in Dodoma. Check stock online.',
  openGraph: { siteName: 'PharmaCare', locale: 'en_TZ', type: 'website' },
  twitter: { card: 'summary_large_image' },
  alternates: { canonical: '/', languages: { en: '/en', sw: '/sw' } },
  robots: { index: true, follow: true },
};
```

```tsx
// app/(public)/medicines/[slug]/page.tsx — per-page dynamic metadata
import type { Metadata } from 'next';

export async function generateMetadata({ params }: PageProps<'/medicines/[slug]'>): Promise<Metadata> {
  const { slug } = await params;
  const m = await getMedicine(slug);            // cached + memoised: the page reuses this call
  if (!m) return { title: 'Medicine not found' };

  return {
    title: `${m.name} — Price & Availability`,
    description: `${m.name}: ${m.price.toLocaleString()} TZS. ${m.stock > 0 ? 'In stock' : 'Currently out of stock'} at PharmaCare Dodoma.`,
    alternates: { canonical: `/medicines/${m.slug}` },
    openGraph: { images: [{ url: m.imageUrl ?? '/og-default.png', width: 1200, height: 630 }] },
  };
}
```

**Social share images:** add `opengraph-image.png` in a route folder, or generate one with code using `ImageResponse` from `next/og` in `opengraph-image.tsx` (medicine name + price on a branded card).

## 23.5 Technical SEO

**Sitemap** — tells search engines every URL you want indexed:

```ts
// app/sitemap.ts → /sitemap.xml
import type { MetadataRoute } from 'next';

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const base = 'https://pharmacare.co.tz';
  const medicines = await getAllMedicines();
  return [
    { url: base, lastModified: new Date(), changeFrequency: 'daily', priority: 1 },
    { url: `${base}/medicines`, changeFrequency: 'daily', priority: 0.9 },
    { url: `${base}/branches`, changeFrequency: 'monthly', priority: 0.6 },
    ...medicines.map((m) => ({ url: `${base}/medicines/${m.slug}`, lastModified: m.updatedAt, priority: 0.8 })),
  ];
}
```

**Robots** — what bots may crawl:

```ts
// app/robots.ts → /robots.txt
export default function robots(): MetadataRoute.Robots {
  return {
    rules: [{ userAgent: '*', allow: '/', disallow: ['/dashboard', '/api', '/login'] }],
    sitemap: 'https://pharmacare.co.tz/sitemap.xml',
  };
}
```

**Technical checklist:**

- **Server-render** public pages (Server Components, static/cached) so content is in the HTML.
- **Canonical URLs** on every page to avoid duplicates (`?sort=price` variants point to the clean URL).
- **Status codes:** real 404s via `notFound()`; **301** (permanent) redirects for moved pages; never redirect everything to the home page.
- **HTTPS** everywhere, one domain version (`www` or not).
- **Mobile-friendly** and fast: Core Web Vitals are a ranking signal (section 24).
- **`noindex`** private or thin pages: `robots: { index: false }` in metadata.
- **`hreflang`** via `alternates.languages` for English/Swahili versions.
- **Pagination and filters:** crawlable links (`<Link href="?page=2">`), not only JS buttons.
- **Breadcrumbs** on deep pages (also as structured data).

## 23.6 Structured data (JSON-LD)

**What it is:** Machine-readable facts about the page using the schema.org vocabulary. It can unlock **rich results** (prices, ratings, opening hours, breadcrumbs in search results) and helps AI systems understand your business.

```tsx
// app/(public)/branches/page.tsx — a local pharmacy
export default function BranchesPage() {
  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'Pharmacy',
    name: 'PharmaCare Dodoma Central',
    url: 'https://pharmacare.co.tz/branches',
    telephone: '+255754000000',
    address: { '@type': 'PostalAddress', streetAddress: 'Jamhuri Street', addressLocality: 'Dodoma', addressCountry: 'TZ' },
    geo: { '@type': 'GeoCoordinates', latitude: -6.163, longitude: 35.7516 },
    openingHoursSpecification: [{ '@type': 'OpeningHoursSpecification', dayOfWeek: ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'], opens: '08:00', closes: '21:00' }],
  };

  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd).replace(/</g, '\\u003c') }}   // escape to prevent XSS
      />
      {/* page content */}
    </>
  );
}
```

```ts
// Medicine page: Product with an Offer
const productLd = {
  '@context': 'https://schema.org',
  '@type': 'Product',
  name: m.name,
  image: m.imageUrl,
  category: m.category.name,
  offers: {
    '@type': 'Offer',
    price: m.price,
    priceCurrency: 'TZS',
    availability: m.stock > 0 ? 'https://schema.org/InStock' : 'https://schema.org/OutOfStock',
  },
};
```

Test with Google's **Rich Results Test** and the Schema.org validator. Mark up only what is visible on the page.

## 23.7 Local SEO (critical for a pharmacy)

- Create and verify a **Google Business Profile** for each branch: correct name, address, phone, hours, photos, category "Pharmacy".
- Keep **NAP** (name, address, phone) identical on the website, Google, Facebook and directories.
- One page per branch with address, map, hours and `Pharmacy` structured data.
- Ask happy customers for Google reviews and reply to reviews.

## 23.8 Off-page SEO and measurement

- **Backlinks:** links from trusted sites (local news, health associations, suppliers, directories) signal authority. Earn them with useful content; never buy them.
- **Google Search Console:** submit the sitemap, see search queries, clicks, indexing problems and Core Web Vitals. Check it weekly.
- **Analytics:** GA4, Plausible or Vercel Analytics for traffic and conversions ("called the branch", "clicked WhatsApp").
- **Monitor:** rankings for key terms, pages losing traffic, crawl errors after deployments.
