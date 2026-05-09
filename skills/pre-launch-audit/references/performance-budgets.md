# Performance Budgets

Reference material for the pre-launch website audit skill. Covers Core Web Vitals thresholds, per-metric diagnostic playbooks, caching strategy, modern browser primitives, and bundle budgets. Written for practitioners who need to diagnose and fix, not just measure.

---

## 1. CWV 2026 Thresholds

Three metrics, all evaluated at the 75th percentile of page loads:

| Metric | Good | Needs Improvement | Poor | What It Measures |
|--------|------|-------------------|------|------------------|
| **LCP** | <= 2.5 s | 2.5 - 4.0 s | > 4.0 s | Time until the largest visible element (image, heading, video poster) finishes rendering |
| **INP** | <= 200 ms | 200 - 500 ms | > 500 ms | Worst-case latency from user interaction (click, tap, key) to the next visual update |
| **CLS** | <= 0.10 | 0.10 - 0.25 | > 0.25 | Cumulative unexpected layout shift over the page's lifetime (session-window scoring) |

**Lab vs. field.** Lighthouse scores are lab measurements: synthetic device, fixed throttling, single page load. CrUX (Chrome User Experience Report) is field data: real users, real devices, real network conditions over 28 days. PageSpeed Insights shows both -- the field section at the top is CrUX, the lab section below is Lighthouse. A pre-launch site has zero field data. Lab numbers are a hypothesis until roughly 28 days of real traffic accumulate post-launch. Plan monitoring accordingly: wire up `web-vitals` JS or a RUM provider (Vercel Analytics, Cloudflare Web Analytics, SpeedCurve) before launch so field data starts collecting on day one.

**Where the web actually stands.** Per the 2025 Web Almanac, 48% of mobile sites pass all three CWV simultaneously. INP is the most-failed metric, with approximately 43% of sites failing. INP replaced FID as the responsiveness vital in March 2024, and many sites that passed FID comfortably are failing INP because INP measures every interaction, not just the first.

**CWV 2.0 / Visual Stability Index (VSI).** Trade press discussed a potential CWV overhaul in early 2026. Google has not confirmed anything. Treat as rumor, not action. The thresholds above are current and stable.

---

## 2. LCP Playbook

LCP failures are almost always one of three things: the hero image is too large, it loads too late, or the server is too slow.

**Identify the LCP element.** Open DevTools, run a Performance recording, and look for the "Largest Contentful Paint" marker in the timeline. Click it to see which DOM element was selected. Common LCP elements: hero images, `<h1>` text with a web font, background images set via CSS, video poster frames.

**Preload the LCP resource.** Once you know the LCP element, ensure the browser discovers it as early as possible. For images, add a preload link in the document `<head>`:

```html
<link rel="preload" as="image" href="/hero.avif" fetchpriority="high">
```

For responsive images, use `imagesrcset` and `imagesizes` on the preload link so the browser selects the right variant without downloading the full set.

**Image format.** AVIF delivers the best compression for photographic content (typically 30-50% smaller than WebP at equivalent quality). WebP is the universal fallback with near-complete browser support. Use a `<picture>` element with AVIF as the primary source and WebP as the fallback:

```html
<picture>
  <source srcset="/hero.avif" type="image/avif">
  <source srcset="/hero.webp" type="image/webp">
  <img src="/hero.jpg" alt="..." width="1200" height="630" fetchpriority="high">
</picture>
```

Always include `srcset` and `sizes` for responsive delivery. A 4000px image served to a 400px viewport is wasted bytes and wasted LCP time.

**Priority hints.** Set `fetchpriority="high"` on the LCP image element. Do not add `loading="lazy"` to above-fold images -- `loading="eager"` is the default and correct behavior for anything the user sees on initial load.

**TTFB.** Target under 600ms for Time to First Byte. If TTFB is higher, the problem is upstream of your images: cold origin, no edge cache, ISR miss, slow database query, or geographic distance between user and server. No amount of image optimization will save an LCP that starts 2 seconds late.

**103 Early Hints.** Cloudflare, Vercel, and Fastly support 103 Early Hints -- the server sends preload hints to the browser before the full HTML response is ready. This lets the browser start fetching the LCP image while the origin is still computing the page. On Vercel, configure via the `Link` header in `next.config.js` or `vercel.json`. On Cloudflare, Early Hints are derived from `Link` headers in previous responses and cached at the edge.

---

## 3. INP Playbook

INP measures the delay between a user action and the browser's visual response. Unlike FID (which only measured the first interaction's input delay), INP captures every interaction and reports near the worst case. A page can have a fast first click and a catastrophically slow menu toggle, and INP will catch the toggle.

**Diagnose.** DevTools Performance panel: look at the "Interactions" track. Each bar shows the full interaction lifecycle -- input delay, processing time, presentation delay. The Long Animation Frames (LoAF) API surfaces frames that took longer than 50ms and attributes the time to specific scripts, making it possible to identify which third-party tag or event handler is responsible.

**Break long tasks.** The browser cannot update the screen while JavaScript holds the main thread. Break expensive operations into smaller chunks:

- `scheduler.postTask()` -- schedule work at different priorities (user-blocking, user-visible, background)
- `requestIdleCallback()` -- defer non-critical work until the browser is idle
- `await new Promise(r => setTimeout(r, 0))` -- the simplest yield pattern; inserts a task boundary so the browser can paint between chunks

**Defer third-party scripts.** Third-party JavaScript is the single largest INP destroyer in 2026. Specific offenders:

- **Chat widgets** (Intercom, Drift, HubSpot Chat): load on interaction (scroll, click), not on page load
- **A/B testing SDKs** (Optimizely, VWO, Google Optimize successor tools): synchronous snippet execution blocks every interaction until the experiment resolves
- **Session recorders** (Hotjar, FullStory, Microsoft Clarity): instrument every DOM mutation, adding overhead to every interaction
- **Tag managers loading 30+ tags**: each tag is a potential long task; audit the container and remove dead tags

For tag managers specifically, consider Partytown -- it moves third-party scripts into a web worker, off the main thread entirely.

**Reduce hydration cost.** Full-page hydration (React client-side hydration of server-rendered HTML) re-executes component trees on load, blocking interactivity. Architecture-level mitigations: React Server Components (zero client JS for server components), Astro islands (only interactive components hydrate), Qwik resumability (no hydration step at all -- event handlers attach lazily).

**Passive listeners.** If you have `pointermove`, `touchmove`, or `scroll` handlers, ensure they are registered as passive (`{ passive: true }`) and throttled. Non-passive handlers prevent the browser from scrolling until the handler completes.

---

## 4. CLS Playbook

Layout shift happens when visible elements move after the user has started reading or interacting. The most common causes are images without dimensions, late-loading fonts, and dynamically injected content.

**Reserve dimensions for media.** Every `<img>` and `<video>` element should have explicit `width` and `height` attributes. The browser uses these to calculate the aspect ratio and reserve space before the resource loads. For embedded content (iframes, tweets, YouTube players), use `aspect-ratio` in CSS:

```css
.embed-container {
  aspect-ratio: 16 / 9;
  width: 100%;
}
```

**Font loading.** Web fonts that arrive late cause a flash of unstyled text (FOUT) or invisible text (FOIT), both of which trigger layout shift. Two strategies:

- `font-display: optional` -- the browser uses the font only if it is already cached; otherwise it uses the fallback permanently. Zero layout shift, but the custom font may not appear on first visit.
- `font-display: swap` combined with `<link rel="preload" as="font" type="font/woff2" href="/font.woff2" crossorigin>` -- the font swaps in when loaded, causing a brief FOUT, but preloading minimizes the window.

Use `optional` for decorative or secondary fonts. Use `swap` + preload for body text where brand typography matters.

**Reserve space for dynamic content.** Cookie consent banners, promotional announcement bars, ad slots, and lazily loaded embeds all inject content into the page after initial render. For each:

- Allocate a fixed-height container in CSS before the content loads
- Position cookie banners and announcements as overlays (fixed/sticky) rather than pushing page content down
- For ad slots, set explicit `min-height` matching the expected ad dimensions

**Animation safety.** Only animate `transform` and `opacity`. These properties are composited on the GPU and do not trigger layout recalculation. Animating `top`, `left`, `width`, `height`, or `margin` causes layout shifts that CLS will count.

---

## 5. Caching Patterns

Poor caching turns every repeat visitor into a first-time visitor from a performance standpoint.

**Fingerprinted static assets.** CSS, JS, images, and fonts with a content hash in the filename (`app.a3b2c1.js`) should be cached aggressively:

```
Cache-Control: public, max-age=31536000, immutable
```

This tells browsers and CDNs to cache for one year and never revalidate. When the content changes, the filename changes, so stale cache is impossible.

**HTML at the edge.** HTML documents change more frequently and need a different strategy:

```
Cache-Control: public, max-age=0, s-maxage=86400, stale-while-revalidate=604800
```

`max-age=0` means browsers always revalidate. `s-maxage=86400` lets the CDN serve from cache for 24 hours. `stale-while-revalidate=604800` lets the CDN serve a stale response while fetching a fresh one in the background for up to 7 days -- users get instant responses, content updates within a day.

**The Vary header.** `Vary: Accept-Encoding` is necessary and harmless -- it tells caches to store separate gzip and brotli variants. `Vary: User-Agent` is catastrophic: it fragments the cache into thousands of variants (one per User-Agent string), reducing CDN hit ratio to near zero. Flag it and remove it. If the intent is to serve different content to mobile vs. desktop, use `Vary: Sec-CH-UA-Mobile` (Client Hints) or handle it at the application layer.

**ISR (Next.js).** Incremental Static Regeneration caches pages at the edge and regenerates them in the background. Verify the `revalidate` value in your page's export makes sense for the content's update frequency. Check the on-demand revalidation route is secured (not publicly callable). Monitor cache state via the `x-nextjs-cache` header: `HIT` (served from cache), `STALE` (served from cache, regenerating), `MISS` (generated on request).

**CDN hit ratio.** Target above 90%. Check in your CDN dashboard (Cloudflare Analytics, Vercel Edge Network, Fastly stats). Below 90% means too many requests are hitting origin, increasing TTFB and server load. Common causes: missing cache headers, overly aggressive `no-cache` directives, `Vary: User-Agent`, and query-string fragmentation.

---

## 6. Modern Primitives

These are browser and platform features that go beyond traditional performance optimization. Not all are universally supported, but where available they provide meaningful gains.

**103 Early Hints.** Supported by Cloudflare, Vercel, and Fastly. The server sends a 103 status code with `Link` headers before the final 200 response. The browser starts preloading critical resources (fonts, CSS, LCP image) while the server is still computing the page. Particularly valuable for dynamic pages with slow TTFB.

**Speculation Rules API.** A declarative way to tell the browser which pages the user is likely to navigate to next. The browser can prefetch or even fully prerender those pages in the background:

```html
<script type="speculationrules">
{
  "prerender": [
    { "where": { "href_matches": "/products/*" }, "eagerness": "moderate" }
  ]
}
</script>
```

Chrome-only as of mid-2026. The prerendered page loads instantly on navigation, making perceived performance near-zero. Check that speculation rules are not breaking analytics (prerendered pages may fire pageview events prematurely) and that the speculation target list is reasonable in size.

**View Transitions API.** Provides smooth, animated transitions between pages in multi-page applications (MPAs) without JavaScript frameworks. Gives the feel of a single-page app with standard navigation. When auditing, verify that view transitions are not breaking back/forward cache eligibility -- some implementations inadvertently add `unload` listeners or manipulate history in ways that disqualify pages.

**Back/forward cache (bfcache).** When a user hits the back button, bfcache serves the previous page from a frozen in-memory snapshot -- instant navigation. DevTools > Application > Back/forward cache shows eligibility and blockers. Common blockers:

- `unload` event listeners (use `pagehide` instead)
- `Cache-Control: no-store` on the HTML response
- Open WebSocket or WebRTC connections
- Pending `IndexedDB` transactions

Fixing bfcache eligibility is often the single highest-impact performance improvement for user-perceived speed, because back-navigation is one of the most common user actions.

---

## 7. Bundle Budgets

**The 170KB threshold.** Any single JavaScript bundle exceeding 170KB gzipped deserves investigation. This is not a hard rule but a practical inflection point: beyond it, parse and compile time on mid-range mobile devices (the 75th percentile device, not a flagship) starts meaningfully impacting INP and LCP.

**Bundle analysis tools.** Use `@next/bundle-analyzer` (Next.js) or `vite-bundle-visualizer` (Vite/Astro/SvelteKit) to see what is inside each bundle. Common findings: entire lodash imported for one function, moment.js still present (replace with date-fns or Temporal), full icon libraries when only 5 icons are used.

**Tree-shaking.** Confirm `"sideEffects": false` in `package.json` for your own code and verify that dependencies support tree-shaking. Without this flag, bundlers assume every module has side effects and cannot eliminate dead code.

**Critical CSS.** Inline the CSS required for above-fold rendering directly in the `<head>`. Defer the rest with `<link rel="stylesheet" href="/styles.css" media="print" onload="this.media='all'">` or equivalent framework mechanisms. This eliminates the render-blocking round trip for the full stylesheet.

**Font strategy.** Self-host fonts rather than loading from Google Fonts. Self-hosting eliminates a DNS lookup, a TLS handshake, and a cross-origin request, and avoids GDPR concerns around Google's data collection. If Google Fonts is unavoidable, add:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```

Use `font-display: swap` for body fonts (users see content immediately in a fallback font) and `font-display: optional` for decorative or heading fonts (skip them entirely if not cached). Subset fonts to the character sets you actually use -- a full Latin Extended set is often 3-4x larger than the basic Latin subset.
