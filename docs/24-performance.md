# 24. Frontend performance

**Why it matters:** many customers browse on mid-range phones over mobile data. A slow page loses them, and Google uses page experience in ranking. Performance is a feature.

## 24.1 Core Web Vitals and other metrics

| Metric | Measures | Good | Usual fixes |
| --- | --- | --- | --- |
| **LCP** (Largest Contentful Paint) | When the main content (hero image, product title) appears | ≤ 2.5 s | Server rendering, fast TTFB, `priority` on the hero image, smaller images |
| **INP** (Interaction to Next Paint) | How fast the page responds to taps and typing | ≤ 200 ms | Less JS, break up long tasks, `useTransition`, fewer re-renders |
| **CLS** (Cumulative Layout Shift) | How much content jumps around while loading | ≤ 0.1 | Set image sizes, reserve space for ads/embeds, `next/font` |
| TTFB (Time to First Byte) | Server response time | ≤ 0.8 s | Caching, CDN, fast queries |
| FCP (First Contentful Paint) | First text or image drawn | ≤ 1.8 s | Less render-blocking CSS/JS |

Scores are judged at the **75th percentile of real users** (field data), not just on your fast laptop.

## 24.2 Measuring

| Tool | Type | Use |
| --- | --- | --- |
| **PageSpeed Insights** | Field (real users, CrUX) + lab | First check for any public URL |
| **Lighthouse** (DevTools tab) | Lab | Audit while developing; use mobile + throttling |
| **Chrome DevTools Performance** | Lab | Find long tasks, layout shifts, slow renders |
| **WebPageTest** | Lab, real devices and locations | Waterfalls, filmstrips, test from Africa-region servers |
| **Search Console → Core Web Vitals** | Field | Site-wide problems Google sees |
| **`web-vitals` / `useReportWebVitals` / Vercel Speed Insights** | Field (your own users) | Track over time |
| **React DevTools Profiler** | Lab | Which components re-render and why |

```tsx
// app/_components/web-vitals.tsx — send real-user metrics to your analytics
'use client';
import { useReportWebVitals } from 'next/web-vitals';
export function WebVitals() {
  useReportWebVitals((metric) => {
    navigator.sendBeacon('/api/vitals', JSON.stringify({ name: metric.name, value: metric.value, page: location.pathname }));
  });
  return null;
}
```

Rule: **measure, fix the biggest problem, measure again.** Set a **performance budget** (e.g. ≤ 150 KB JS per route, LCP ≤ 2.5 s on Slow 4G) and check it in CI.

## 24.3 Images (usually the biggest win)

- Use `next/image`: automatic resizing, **WebP/AVIF**, lazy loading, and width/height to stop layout shift.
- Set `sizes` so phones download small versions.
- Add `priority` only to the above-the-fold (LCP) image; everything else lazy-loads.
- Compress source images; never upload 4000 px photos for a 400 px card.
- Use SVG for logos and icons; prefer an icon library that tree-shakes (lucide-react).

## 24.4 Fonts

- `next/font`: self-hosted, preloaded, with automatic size-adjusted fallbacks (no layout shift).
- Use one or two families, few weights; prefer **variable fonts**; subset to the characters you need (`subsets: ['latin']`).
- `display: 'swap'` shows text immediately in a fallback font.

## 24.5 JavaScript

- **Ship less JS:** keep components on the server; `'use client'` only on interactive leaves.
- **Code splitting:** Next.js splits per route automatically; load heavy widgets (charts, PDF viewer, rich-text editor) with `dynamic()` or `lazy()`.
- **Tree-shaking:** use ES-module libraries and named imports; check bundles with the analyser.
- **Avoid long tasks** (> 50 ms) on the main thread: split work, use `useTransition`, move heavy calculations to a **Web Worker**, virtualise long lists.
- **Debounce** search inputs; avoid unnecessary re-renders (React Compiler, state close to where it's used).
- **Minification and compression** (Brotli/gzip) are automatic with `next build` and most hosts.

## 24.6 CSS

- Tailwind ships only used classes (small CSS).
- Avoid huge CSS frameworks you barely use; avoid CSS-in-JS runtimes on the critical path.
- Animate only `transform` and `opacity` (cheap, GPU-accelerated); avoid animating `width`, `top`, `height`.

## 24.7 Network and caching

- **Cache on the server:** `'use cache'` for shared data; static pages served from a CDN.
- **HTTP caching:** long `Cache-Control` with content hashes for static assets (Next.js does this for `/_next/static`), short or no cache for HTML that changes.
- **CDN** close to users; enable HTTP/2 or HTTP/3.
- **Resource hints:** `<link rel="preconnect">` to critical third-party origins; `<Link>` prefetches routes automatically.
- **Reduce requests and payload size:** return only the fields the page needs from the API; paginate lists.
- **Service workers** (PWA, section 26) can cache assets and API responses for offline use.

## 24.8 Rendering strategy

- Static or cached pages are the fastest (no server work per request).
- Stream slow parts with `<Suspense>` so the shell appears immediately.
- Avoid request waterfalls: fetch in parallel; preload data.
- Keep the DOM reasonably small (huge tables → pagination or virtualisation).

## 24.9 Third-party scripts

Analytics, chat widgets, ads and tag managers are a top cause of slow INP and LCP.

- Audit what you really need; remove unused tags.
- Load with `next/script` `strategy="lazyOnload"` or `afterInteractive`; use `@next/third-parties` for Google tools.
- Facade pattern: show a static preview image of a YouTube video or map, load the real embed on click.

## 24.10 Performance checklist

- [ ] Lighthouse mobile performance ≥ 90 on key pages; LCP ≤ 2.5 s, INP ≤ 200 ms, CLS ≤ 0.1 in field data.
- [ ] All images via `next/image` with correct `sizes`; LCP image has `priority`.
- [ ] Fonts via `next/font`; at most two families.
- [ ] No `'use client'` on whole pages or layouts unless needed.
- [ ] Heavy components loaded dynamically; bundle analysed; no duplicate libraries (e.g. two date libraries).
- [ ] Long lists paginated or virtualised.
- [ ] Data fetched in parallel; slow sections wrapped in `<Suspense>`.
- [ ] Shared data cached; static pages served from a CDN.
- [ ] Third-party scripts deferred or removed.
- [ ] Real-user Web Vitals tracked and reviewed monthly.
