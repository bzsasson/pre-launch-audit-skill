# Performance Audit Playbook

Sub-audit module for the Pre-Launch Website Audit skill. Covers Core Web Vitals, Lighthouse scoring, network analysis, image optimization, caching, render-blocking resources, modern performance primitives, and bundle analysis.

**Primary tools:** Chrome DevTools (`lighthouse_audit`, `performance_start_trace`/`stop`, `list_network_requests`, `evaluate_script`), DataForSEO `on_page_lighthouse`
**Fallback:** DataForSEO Lighthouse API returns LCP, CLS, FCP, scores, and byte weight without requiring a browser session. Bash curl for header inspection and cache analysis.

---

## Pre-Launch Caveat

A pre-launch site has no CrUX field data. Every number produced in this audit is a lab measurement -- a hypothesis, not a verdict. Field data requires approximately 28 days of real-user traffic to accumulate in CrUX. State this explicitly in the report and recommend wiring Real User Monitoring (Vercel Analytics, Cloudflare Web Analytics, or the `web-vitals` JS library) before launch so field data starts collecting on day one.

Lab and field numbers diverge for predictable reasons: lab tests use fixed network conditions and device profiles; field data reflects the full distribution of user hardware, connections, and interaction patterns. INP in particular is almost impossible to measure accurately in lab conditions because it depends on actual user interactions.

---

## 1. Core Web Vitals (2026 Thresholds)

### Thresholds

| Metric | Good | Needs Improvement | Poor | What It Measures |
|--------|------|--------------------|------|------------------|
| **LCP** | <= 2.5s | 2.5 -- 4.0s | > 4.0s | Largest visible element render time |
| **INP** | <= 200ms | 200 -- 500ms | > 500ms | Responsiveness to user interactions |
| **CLS** | <= 0.10 | 0.10 -- 0.25 | > 0.25 | Visual stability during loading and interaction |

All metrics are evaluated at the 75th percentile of page loads. Both mobile and desktop viewports must be tested. Mobile is the priority viewport -- Google completed the full mobile-first indexing migration on July 5, 2024.

INP replaced FID as the responsiveness Core Web Vital on March 12, 2024. It is the most-failed metric: approximately 43% of sites fail it according to the 2025 Web Almanac. Only 48% of mobile pages pass all three CWV.

### Tool invocations

**Chrome DevTools performance trace (preferred):**
```
call_mcp_tool(mcp_name='chrome-devtools', tool_name='performance_start_trace', arguments={})
```
Navigate to the target page and interact with key elements (click buttons, open menus, scroll), then:
```
call_mcp_tool(mcp_name='chrome-devtools', tool_name='performance_stop_trace', arguments={})
```
The trace provides element-level attribution: which specific DOM element is the LCP element, which interaction triggered the worst INP, and which elements shifted to cause CLS.

**DataForSEO fallback:**
```
call_mcp_tool(mcp_name='dataforseo', tool_name='on_page_lighthouse', arguments={"url": "https://example.com", "for_mobile": true})
```
Returns LCP, CLS, FCP, performance score, and byte weight. Does not return INP (requires real interaction). Run a second call with `"for_mobile": false` for desktop comparison.

### LCP mini-playbook

1. **Identify the LCP element.** Use the performance trace or DevTools Performance panel. Common LCP elements: hero image, above-the-fold heading, background image via CSS.
2. **Preload it.** Add `<link rel="preload" as="image" href="hero.avif" fetchpriority="high">` in the document head. For responsive images, use `imagesrcset` and `imagesizes` attributes on the preload link.
3. **Serve modern formats.** AVIF offers the best compression. WebP is the universal fallback. Use `<picture>` with format fallbacks. Never serve a 4000px source to a 400px viewport -- use `srcset` and `sizes`.
4. **Set fetchpriority.** `fetchpriority="high"` on the LCP image element. Ensure `loading="eager"` (the default for above-fold images -- do NOT set `loading="lazy"` on the LCP image).
5. **Reduce TTFB.** Target under 600ms. If higher, the platform is the bottleneck: origin server too far from users, no edge cache, ISR cold start. Check `x-vercel-cache`, `cf-cache-status`, or `x-nextjs-cache` headers for cache state.
6. **103 Early Hints.** On Cloudflare, Vercel, or Fastly, configure Early Hints to send preload directives before the main response is ready. This lets the browser start fetching the LCP resource while the server is still computing the HTML.

### INP mini-playbook

1. **Diagnose with Long Animation Frames (LoAF).** In DevTools Performance panel, look at the "Interactions" track. LoAF entries show which scripts ran during the interaction and how long they took.
2. **Break long tasks.** No single task should block the main thread for more than 50ms. Split expensive work:
   - `scheduler.postTask()` for priority-aware scheduling
   - `requestIdleCallback()` for non-urgent work
   - `await new Promise(r => setTimeout(r, 0))` as a simple yield point between synchronous operations
3. **Defer third-party JS.** Chat widgets, A/B test SDKs, and session recorders (Hotjar, FullStory) are the most common INP killers in 2026. Use Partytown to run tag manager scripts in a web worker. Lazy-load chat widgets on user interaction, not on page load.
4. **Reduce hydration cost.** This is an architecture-level fix. React Server Components eliminate client-side hydration for server-rendered content. Astro islands hydrate only interactive components. Qwik uses resumability to avoid hydration entirely. If the framework does not support partial hydration, at minimum defer non-critical component hydration.
5. **Audit event listeners.** Passive `pointermove` handlers without throttling, `scroll` listeners without `requestAnimationFrame` gating, and `input` handlers doing synchronous work all degrade INP.

### CLS mini-playbook

1. **Reserve dimensions for all media.** Set explicit `width` and `height` attributes on every `<img>` and `<video>` element. Use `aspect-ratio` in CSS for embeds and iframes.
2. **Control font loading.** Use `font-display: swap` for body text (prevents invisible text) and `font-display: optional` for decorative fonts (prevents late-arriving reflow). Preload critical font files.
3. **Reserve space for dynamic content.** Ads, cookie consent banners, notification bars, and "above the fold" announcements that inject content after load are common CLS sources. Allocate fixed dimensions in CSS before the content loads.
4. **Animate only transform and opacity.** Layout-triggering properties (width, height, top, left, margin, padding) cause shifts when animated. Stick to `transform` and `opacity` for all animations.
5. **Watch for late-injected content.** Any script that inserts DOM elements above existing visible content after initial render causes CLS. This includes A/B test variations that swap hero banners and lazy-loaded components that push content down.

### Consultant interpretation example

> LCP is 3.8s on mobile (needs improvement). The LCP element is a 2.4MB PNG hero image served without preload or fetchpriority. Three fixes bring this under 2.5s: convert to AVIF (estimated ~180KB), add `<link rel="preload" as="image" fetchpriority="high">`, and confirm CDN cache hit (currently showing `cf-cache-status: MISS` on first load). TTFB is 420ms, which is healthy -- the bottleneck is image weight and discovery time, not server response.
>
> INP is 340ms (needs improvement). The trace shows a chat widget initialization running 280ms of synchronous JS on every click interaction. Deferring the widget load to first user interaction (scroll or click) would eliminate this. Separately, a third-party A/B SDK adds 60ms to every interaction via a synchronous event listener.
>
> CLS is 0.04 (good). No action needed.

---

## 2. Lighthouse Scores

Run Lighthouse for both mobile and desktop viewports. The audit produces four category scores:

| Category | Target | Notes |
|----------|--------|-------|
| **Performance** | >= 90 | Use perf trace for CWV detail; Lighthouse performance score is a weighted composite |
| **Accessibility** | >= 90 | Score < 90 is a near-perfect proxy for agent-hostility (cross-connects to AI Accessibility audit) |
| **Best Practices** | >= 90 | Flags HTTPS issues, deprecated APIs, console errors |
| **SEO** | >= 90 | Basic meta/crawlability checks; not a substitute for full technical SEO audit |

**Chrome DevTools Lighthouse:**
```
call_mcp_tool(mcp_name='chrome-devtools', tool_name='lighthouse_audit', arguments={})
```
Note: the Chrome DevTools `lighthouse_audit` tool excludes the Performance category. Use the performance trace (section 1) for CWV numbers.

**DataForSEO Lighthouse:**
```
call_mcp_tool(mcp_name='dataforseo', tool_name='on_page_lighthouse', arguments={"url": "https://example.com", "for_mobile": true})
```
DataForSEO's Lighthouse includes performance metrics (LCP, CLS, FCP, byte weight). Run both tools and compare: Chrome DevTools gives richer diagnostics; DataForSEO gives a quick performance snapshot without browser setup.

Report mobile scores first (mobile-first indexing is complete). Flag any category below 90 as P1. Flag Performance below 50 as P0.

---

## 3. Network Waterfall

Use Chrome DevTools to capture the full network waterfall:
```
call_mcp_tool(mcp_name='chrome-devtools', tool_name='list_network_requests', arguments={})
```

### What to check

**TTFB (Time to First Byte).** Target under 600ms. TTFB above 800ms on a CDN-fronted site indicates a cache miss, cold ISR start, or origin server problem. TTFB above 1.5s is a P0 -- no amount of front-end optimization compensates for a slow server.

**Total page weight.** Sum all transferred bytes. Flag pages exceeding 3MB total (mobile on 4G). Identify the top 5 largest resources by size.

**Request count.** Flag pages with more than 80 requests. High request counts indicate missing bundling, excessive third-party scripts, or unoptimized asset loading.

**Third-party script inventory.** List every domain that is not the site's own. Common third-party categories and their typical impact:
- Analytics (GA4, Plausible): usually < 30KB, low impact
- Tag managers (GTM): low weight but can load dozens of downstream scripts
- Chat widgets (Intercom, Drift, Zendesk): 200-500KB, often INP killers
- A/B testing (Optimizely, VWO, LaunchDarkly): render-blocking if synchronous
- Session recorders (Hotjar, FullStory): 100-300KB, heavy on INP
- Social embeds (Twitter/X, Instagram): unpredictable weight, CLS risk

For each third-party script, note whether it is loaded synchronously (render-blocking) or asynchronously. Flag synchronous third-party scripts as P1.

**Resource hints.** Check the document head for:
- `<link rel="preload">` -- critical resources needed for first render
- `<link rel="preconnect">` -- early connection to third-party origins
- `<link rel="dns-prefetch">` -- lightweight DNS resolution for less-critical origins

Missing preconnect for frequently-used third-party origins (fonts, CDN, analytics) is a P2 finding.

---

## 4. Image Optimization

### Format

Check every image resource in the network waterfall. Flag images served as PNG or JPEG where WebP or AVIF would be appropriate. Exceptions: PNG is correct for images requiring transparency with sharp edges (logos, icons); JPEG is acceptable when WebP/AVIF is not supported by the serving infrastructure.

Use `<picture>` with format fallbacks:
```html
<picture>
  <source srcset="hero.avif" type="image/avif">
  <source srcset="hero.webp" type="image/webp">
  <img src="hero.jpg" alt="..." width="1200" height="630">
</picture>
```

### Lazy loading

Every below-fold image should have `loading="lazy"`. The LCP image must NOT be lazy-loaded -- it must be `loading="eager"` (the default) with `fetchpriority="high"`.

Check with:
```
call_mcp_tool(mcp_name='chrome-devtools', tool_name='evaluate_script', arguments={"expression": "Array.from(document.querySelectorAll('img')).map(i => ({src: i.src.substring(0, 80), loading: i.loading, fetchpriority: i.fetchPriority, width: i.width, height: i.height, inViewport: i.getBoundingClientRect().top < window.innerHeight}))"})
```

### Dimensions for CLS prevention

Every `<img>` and `<video>` element must have explicit `width` and `height` attributes (or CSS `aspect-ratio`). Missing dimensions cause layout shifts when images load. This is one of the most common CLS sources.

### LCP image requirements

The LCP image has specific requirements beyond general image optimization:
1. **Preloaded:** `<link rel="preload" as="image" href="..." fetchpriority="high">` in the document head
2. **CDN-cached:** verify the image URL returns a cache hit header (`cf-cache-status: HIT`, `x-vercel-cache: HIT`, etc.)
3. **Eager loading:** `loading="eager"` (the default -- just do not set `loading="lazy"`)
4. **High fetch priority:** `fetchpriority="high"` on the `<img>` element
5. **Not CSS background:** CSS background images are discovered later than `<img>` elements; prefer `<img>` for LCP

Flag any LCP image violation as P1 (directly impacts the most visible CWV metric).

---

## 5. Caching & CDN

### Static asset caching

Fingerprinted static assets (JS bundles, CSS files, font files, images with content hashes in filenames) should have aggressive cache headers:
```
Cache-Control: public, max-age=31536000, immutable
```

`immutable` tells the browser not to revalidate even on reload. This is correct only for fingerprinted assets where the filename changes when content changes.

Non-fingerprinted assets (HTML, API responses) need different caching:
```
Cache-Control: public, max-age=0, s-maxage=86400, stale-while-revalidate=604800
```

Check cache headers with bash:
```bash
curl -sI https://example.com/ | grep -i cache-control
curl -sI https://example.com/_next/static/chunks/main-abc123.js | grep -i cache-control
curl -sI https://example.com/styles.css | grep -i cache-control
```

### CDN hit/miss headers

Check CDN cache status on multiple requests to the same URL. Key headers by platform:

| Header | Platform | Values |
|--------|----------|--------|
| `cf-cache-status` | Cloudflare | HIT, MISS, EXPIRED, DYNAMIC, BYPASS |
| `x-vercel-cache` | Vercel | HIT, MISS, STALE, PRERENDER |
| `x-nextjs-cache` | Next.js ISR | HIT, MISS, STALE |
| `x-nf-request-id` | Netlify | (presence confirms Netlify; check `x-nf-cache-status`) |
| `x-cache` | CloudFront/generic | Hit from cloudfront, Miss from cloudfront |
| `age` | Any | Seconds since cached; 0 = fresh cache or no cache |

Target: CDN hit ratio above 90% for static assets. A site where most requests show MISS or DYNAMIC has a caching configuration problem.

### The `Vary: User-Agent` trap

`Vary: User-Agent` in response headers tells caches to store a separate copy for every distinct User-Agent string. Since User-Agent strings are effectively infinite (browser version, OS version, device model), this fragments the cache catastrophically -- the hit ratio drops toward zero.

Flag `Vary: User-Agent` as P1. The fix is to remove it. If the site needs to serve different content to different devices, use `Vary: Accept` (for format negotiation) or client hints (`Sec-CH-UA-Mobile`) instead.

`Vary: Accept-Encoding` is correct and expected (serves gzip vs brotli vs uncompressed).

### ISR verification (Next.js)

If the stack is Next.js with ISR:
1. Check `revalidate` values in page components -- are they appropriate for content freshness needs?
2. Verify on-demand revalidation route exists (`/api/revalidate` or equivalent)
3. Request a page twice. First request may show `x-nextjs-cache: MISS` or `STALE`; second should show `HIT`
4. If `x-nextjs-cache: STALE` persists across requests, revalidation is failing

### Consultant interpretation example

> Static assets are correctly fingerprinted with `max-age=31536000, immutable` -- well configured. However, the HTML document returns `Cache-Control: no-store`, which prevents CDN edge caching entirely and forces every request back to origin. This explains the 1.2s TTFB: the origin is in us-east-1 and your European users are hitting it directly. Switching to `s-maxage=3600, stale-while-revalidate=86400` on the CDN layer would bring TTFB under 100ms for cached pages while still serving fresh content within an hour.
>
> Additionally, `Vary: User-Agent` is present on all HTML responses. This fragments the edge cache -- each of the hundreds of browser/OS combinations gets its own cached copy, so the effective hit rate is near zero. Remove this header. If device-specific content is needed, use `Sec-CH-UA-Mobile` with `Vary: Sec-CH-UA-Mobile` (two cache variants instead of hundreds).

---

## 6. Render-Blocking Resources

### CSS in the document head

All `<link rel="stylesheet">` tags in `<head>` block first paint. Check:
- Is critical CSS inlined in a `<style>` tag? (Ideal for above-fold content)
- Are non-critical stylesheets deferred with `media="print" onload="this.media='all'"` or loaded via JS?
- How large is the total blocking CSS? Flag if total blocking CSS exceeds 100KB uncompressed.

### Synchronous JavaScript

Any `<script>` tag in `<head>` without `async` or `defer` blocks rendering. Check:
```
call_mcp_tool(mcp_name='chrome-devtools', tool_name='evaluate_script', arguments={"expression": "Array.from(document.querySelectorAll('head script[src]:not([async]):not([defer]):not([type=\"module\"])')).map(s => s.src)"})
```

Flag synchronous scripts in `<head>` as P1. Common offenders: analytics snippets, A/B test loaders, consent management platforms.

`<script type="module">` is deferred by default and does not block rendering.

### Font loading

**`font-display` values:**
- `swap` -- for body/primary fonts. Shows fallback text immediately, swaps when font loads. Prevents invisible text (FOIT).
- `optional` -- for decorative/secondary fonts. Uses the font only if it arrives within ~100ms; otherwise keeps the fallback permanently. Eliminates late-arriving font reflow (prevents CLS).
- `block` -- the default and the worst. Hides text for up to 3 seconds waiting for the font. Flag as P1.
- `fallback` -- similar to `optional` but with a slightly longer swap window. Acceptable.

Check font-display values:
```bash
curl -sL https://example.com | grep -oP 'font-display:\s*\w+'
```

**Self-hosting vs Google Fonts.** Self-hosting fonts is preferred for both performance and GDPR compliance (Google Fonts makes a request to Google's servers, which has triggered GDPR complaints in the EU). If the site uses Google Fonts:
1. Verify `<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>` is present
2. Also add `<link rel="preconnect" href="https://fonts.googleapis.com">`
3. Recommend migrating to self-hosted fonts as a P2 improvement

Check for self-hosted vs external fonts:
```
call_mcp_tool(mcp_name='chrome-devtools', tool_name='evaluate_script', arguments={"expression": "Array.from(document.querySelectorAll('link[href*=\"fonts.googleapis\"], link[href*=\"fonts.gstatic\"], link[href*=\"use.typekit\"]')).map(l => l.href)"})
```

---

## 7. Modern Primitives

These are P3/opportunity-level findings. They represent the leading edge of web performance and are not required for launch. Flag as opportunities, not blockers.

### 103 Early Hints

Early Hints let the server send preload/preconnect directives in an interim 103 response before the full 200 response is ready. The browser starts fetching critical resources while the server is still computing the page.

Supported by: Cloudflare (automatic for preload/preconnect Link headers), Vercel (via `headers()` in Next.js), Fastly.

Check if Early Hints are configured:
```bash
curl -sI --http2 https://example.com | head -30
```
Look for a 103 status code before the 200. Most curl versions do not display 103 responses -- note this limitation in the report if you cannot confirm.

### Speculation Rules API

`<script type="speculationrules">` enables prefetch or prerender of likely next pages. Works in Chrome (Chromium-based browsers). Example:
```html
<script type="speculationrules">
{
  "prerender": [{"where": {"href_matches": "/product/*"}, "eagerness": "moderate"}],
  "prefetch": [{"where": {"href_matches": "/*"}, "eagerness": "conservative"}]
}
</script>
```

Check if present:
```
call_mcp_tool(mcp_name='chrome-devtools', tool_name='evaluate_script', arguments={"expression": "document.querySelector('script[type=\"speculationrules\"]')?.textContent"})
```

If absent, recommend as P3 for navigation-heavy sites (e-commerce, content publishers). The performance gain on subsequent page loads is significant -- near-instant navigation.

### View Transitions API

Enables animated transitions between page navigations in multi-page apps. Check that View Transitions are not breaking back/forward cache (bf-cache). If `document.startViewTransition()` is used, verify it does not register `unload` listeners or set `Cache-Control: no-store`, both of which break bf-cache.

### Bf-cache eligibility

Back/forward cache (bf-cache) lets the browser instantly restore pages when the user clicks Back or Forward. Common blockers:

- `unload` event listeners (use `pagehide` instead)
- `Cache-Control: no-store` on the document
- `beforeunload` listeners (acceptable only when form data is present)
- `window.opener` references (pages opened via `window.open`)

Check eligibility:
```
call_mcp_tool(mcp_name='chrome-devtools', tool_name='evaluate_script', arguments={"expression": "typeof window.onunload !== 'undefined' ? 'unload listener detected' : 'no unload listener'"})
```

DevTools Application panel > Back/forward cache provides a full eligibility report. If available, use it. Otherwise, flag known blockers from code inspection.

Bf-cache failure is P2 for content sites (users frequently navigate back) and P3 for single-page apps (where client-side routing handles back navigation).

---

## 8. Bundle Analysis

This section applies only when auditing a local project with access to the source code and build output.

### Bundle size threshold

The combined JavaScript payload should not exceed 170KB gzipped for the initial page load. This includes framework runtime, application code, and synchronous third-party scripts. It excludes lazy-loaded chunks and deferred scripts.

Check the build output:
```bash
# Next.js
ls -lhS .next/static/chunks/*.js | head -20

# Vite/Astro/SvelteKit
ls -lhS dist/assets/*.js 2>/dev/null || ls -lhS dist/_astro/*.js 2>/dev/null | head -20

# Gzipped sizes
find .next/static/chunks -name '*.js' -exec sh -c 'echo "$(gzip -c "$1" | wc -c) $1"' _ {} \; | sort -rn | head -10
```

### Bundle analyzers

- **Next.js:** `@next/bundle-analyzer` -- add to `next.config.js`, run `ANALYZE=true npm run build`
- **Vite (Astro, SvelteKit, etc.):** `vite-bundle-visualizer` or `rollup-plugin-visualizer`
- **Webpack (general):** `webpack-bundle-analyzer`

Flag any single chunk exceeding 100KB gzipped. Common culprits: moment.js (use date-fns or dayjs), lodash (use lodash-es with tree-shaking), full icon libraries (import individual icons).

### Tree-shaking

Verify `"sideEffects": false` in `package.json` (or a specific list of side-effect files). Without this declaration, bundlers cannot safely tree-shake unused exports.

Check:
```bash
cat package.json | grep -A2 sideEffects
```

### Critical CSS

For the initial page load, only CSS needed for above-the-fold content should be render-blocking. The rest should be deferred. Tools like `critters` (used by Angular CLI and available as a Vite plugin) extract and inline critical CSS automatically.

### Third-party JS impact assessment

For each third-party script identified in the network waterfall (section 3), assess:

| Script type | Typical weight (gzipped) | INP risk | Recommendation |
|-------------|--------------------------|----------|----------------|
| Chat widgets (Intercom, Drift, Zendesk) | 200-500KB | High | Lazy-load on first interaction |
| A/B testing (Optimizely, VWO) | 50-150KB | Medium-High | Load async, never synchronous |
| Session recorders (Hotjar, FullStory) | 100-300KB | High | Defer to post-load or sample |
| Analytics (GA4, Plausible) | 20-50KB | Low | Async script, low concern |
| Consent management (OneTrust, Cookiebot) | 50-200KB | Medium | Required by law, optimize config |
| Social embeds (Twitter, Instagram) | Variable | Medium | Lazy-load, reserve dimensions |
| Tag managers (GTM) | 30-80KB base | Varies | Audit loaded tags -- the container itself is small but what it loads matters |

Chat widgets, A/B SDKs, and session recorders are the most common INP killers. If INP is failing, these are the first suspects. Recommend Partytown for offloading non-critical third-party scripts to a web worker.

---

## Severity Assignment

| Finding | Default Severity | Rationale |
|---------|------------------|-----------|
| LCP > 4.0s | P0 | Directly impacts search ranking (CWV signal) |
| INP > 500ms | P0 | Most-failed CWV metric; users feel unresponsive pages immediately |
| CLS > 0.25 | P1 | Annoying but less likely to cause bounce than LCP/INP failure |
| Lighthouse Performance < 50 | P0 | Indicates fundamental performance problems |
| TTFB > 1.5s | P0 | Server-side problem; no front-end fix compensates |
| TTFB 600ms -- 1.5s | P1 | Investigate caching and CDN configuration |
| `Vary: User-Agent` on HTML | P1 | Fragments CDN cache, negates edge caching |
| Synchronous third-party JS in head | P1 | Blocks rendering for all users |
| LCP image not preloaded | P1 | Easy fix with high impact |
| Images without width/height | P1 | Causes CLS on every page load |
| Missing `font-display` or set to `block` | P1 | Invisible text for up to 3 seconds |
| PNG/JPEG hero image (WebP/AVIF available) | P2 | Weight reduction opportunity |
| No `preconnect` for third-party origins | P2 | Saves 100-300ms per origin |
| Bundle > 170KB gzipped | P2 | Impacts parse/compile time on low-end devices |
| No Speculation Rules | P3 | Opportunity for faster subsequent navigations |
| No Early Hints | P3 | Opportunity for faster LCP on supported platforms |
| Bf-cache ineligible | P2-P3 | Depends on site navigation patterns |

---

## Cross-Connections to Other Sub-Audits

Performance findings frequently connect to other audit domains. Surface these connections in the synthesis phase:

- **LCP image not CDN-cached** connects to **Caching & CDN** (this audit) and **Technical SEO** (ISR/SSG cache validation)
- **Synchronous third-party JS** connects to **Security** (SRI on CDN scripts, supply chain risk) and **On-Page** (INP impact on user engagement metrics)
- **`font-display: block`** connects to **On-Page** (CLS impact on visual experience) and **AI Accessibility** (text invisible during FOIT is invisible to agentic browsers too)
- **Missing image dimensions** connects to **On-Page** (image SEO, alt text audit) and overlaps with CLS findings
- **`Cache-Control: no-store`** connects to **Modern Primitives** (breaks bf-cache) and **Caching** (prevents edge caching)
- **CSP blocks inline scripts** connects to **Security** (CSP is correct) but may break performance optimizations (inline critical CSS, schema injection)

Report each cross-connection once, in whichever sub-audit it has the highest severity. Reference the other sub-audit finding by name so the reader can trace the connection.

---

## Post-Launch Monitoring Setup

Before launch, wire the following so performance data starts collecting immediately:

1. **Real User Monitoring (RUM).** Deploy one of: Vercel Analytics (if on Vercel), Cloudflare Web Analytics (if on Cloudflare), or the `web-vitals` JS library reporting to your analytics platform.
2. **Search Console Core Web Vitals report.** Verify the property in GSC. CWV data appears after approximately 28 days of traffic.
3. **PageSpeed Insights / CrUX.** Bookmark the PSI URL for key pages. Field data appears at the top of the report once CrUX has sufficient data.
4. **Lighthouse CI.** Set up `@lhci/cli` in CI with a budget JSON file. Fail builds on performance regression beyond defined thresholds.
5. **CDN analytics.** Enable cache analytics in the CDN dashboard (Cloudflare Analytics, Vercel Edge Network, CloudFront reports) to monitor cache hit ratio over time. Target: sustained > 90%.
6. **Error monitoring.** Wire Sentry or equivalent to catch client-side JavaScript errors that may indicate broken interactions (contributing to INP failures in the field).
