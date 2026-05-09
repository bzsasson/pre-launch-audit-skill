# Technical SEO Audit Playbook

**Sub-audit 1 of 5** | Pre-Launch Website Audit Skill
**Primary tools:** SF MCP, Chrome DevTools, DataForSEO `on_page_instant_pages`
**Fallback:** DataForSEO on-page + bash curl spot-checks

This playbook covers the 10 technical SEO checks in priority order. Every check explains the business consequence of failure, provides exact tool invocations, and defines what good and bad look like. Stack-specific variations are called out inline.

---

## 1. Indexation Blockers

**Priority: P0 -- the #1 pre-launch killer.**

A single stray `noindex` directive left from staging can make an entire site invisible to every search engine and AI crawler the moment it goes live. Unlike a slow page or a missing description, an indexation blocker produces zero organic traffic -- not reduced traffic, zero. You will not see an error. You will not get a warning. The site will simply not exist in search results, and nobody will notice until someone asks "why aren't we ranking for anything?" two weeks later.

### What to check

Four directive types can block indexation:

- **noindex** -- the explicit "do not index this page" signal, delivered via `<meta name="robots" content="noindex">` or `X-Robots-Tag: noindex` HTTP header. The header is especially dangerous because it is invisible in the HTML source and applies to non-HTML resources (PDFs, images).
- **none** -- equivalent to `noindex, nofollow` combined. Often set by CMS defaults on staging or by overzealous security plugins.
- **nofollow** -- does not block indexation directly, but a sitewide nofollow prevents link equity from flowing through the site's internal architecture, effectively orphaning every page beyond the homepage from a ranking perspective.
- **robots.txt Disallow** -- prevents crawling entirely. A page blocked by robots.txt cannot be indexed (with one exception: Google may index the URL without content if it finds external links pointing to it, creating a thin, titleless SERP listing -- worse than not appearing at all).

### Tool invocations

**Screaming Frog (primary):**

```
Export: Directives > Directives:Noindex
Export: Directives > Directives:None
Export: Directives > Directives:Nofollow
Export: Response Codes > Response Codes:Blocked by Robots.txt
```

These four exports cover every mechanical indexation blocker. Run all four and cross-reference against the intended URL set.

**Bash (fallback or spot-check):**

```bash
# Check meta robots on a specific URL
curl -sL https://example.com/ | grep -i 'name="robots"'

# Check X-Robots-Tag header
curl -sI https://example.com/ | grep -i 'x-robots-tag'

# Check robots.txt for broad blocks
curl -s https://example.com/robots.txt | grep -E '^Disallow:\s*/'
```

**DataForSEO (when SF unavailable):**

Use `on_page_instant_pages` with the target URL. The response includes `meta.robots` and `checks.is_noindex` fields.

### What good looks like

- Zero pages in the Noindex, None, or Blocked by Robots.txt exports that are intended for indexation.
- Nofollow used only on specific outbound links (e.g., sponsored, UGC), never sitewide.
- `robots.txt` has targeted `Disallow` rules for admin, API, and staging paths only, not broad wildcards that catch content pages.

### What bad looks like

- Any page in the primary navigation or sitemap appearing in the Noindex export.
- `X-Robots-Tag: noindex` set at the server or CDN level (common in staging environments with Vercel preview deployments or Netlify branch deploys).
- `Disallow: /` in robots.txt -- the staging leftover that blocks everything.
- A `none` directive anywhere in production templates.

### Consultant interpretation example

> "Your robots.txt contains `Disallow: /` under `User-agent: *`, which blocks every crawler from accessing any page on the site. This is almost certainly a staging configuration that was carried over to production. Until this is removed, Google, Bing, and every AI search engine will treat your site as if it does not exist. This is a P0 launch blocker -- the single highest-priority fix in this audit. Remove the line, verify with `curl -s https://example.com/robots.txt`, and confirm via Google Search Console's robots.txt tester."

### Stack-specific notes

- **Vercel preview deployments** set `X-Robots-Tag: noindex` by default. If the production domain is misconfigured to point to a preview deployment instead of the production deployment, the entire site is noindexed via an invisible header.
- **Next.js** generates a default `robots.ts` or `robots.js` in the App Router; verify the production output, not just the source file.
- **WordPress** has a "Discourage search engines from indexing this site" checkbox under Settings > Reading. It sets a sitewide noindex. This is the most common WordPress indexation blocker after migration.
- **Shopify** adds `noindex` to paginated collection pages beyond page 1 by default and to certain filtered views.

---

## 2. JS Rendering and Crawlability

**Priority: P0-P1 depending on stack.**

Search engines render JavaScript, but not all of them, not all the time, and not instantly. Googlebot uses an evergreen Chromium renderer with a queue and a soft 5-second budget per page. Bingbot also renders. But the AI crawlers that feed ChatGPT Search, Claude, and Perplexity behave differently: GPTBot and ClaudeBot training crawlers historically do not execute JavaScript, though this is changing in 2026. OAI-SearchBot and PerplexityBot execute JS in some configurations but at lower fidelity than Googlebot.

The business consequence: if your content exists only in the rendered DOM (client-side rendered, hydrated from an API, or injected by a JavaScript framework after initial page load), some portion of the crawlers that matter -- including the ones that power AI search citations -- may see a blank page.

### What to check

**Rendered vs raw HTML diff** -- the foundational JS SEO check. If the raw HTML (what `curl` returns) is missing content that appears in the rendered DOM (what a browser with JavaScript sees), that content is at risk.

**JS-only internal links** -- links injected by JavaScript after page load are invisible to crawlers that do not render. This fragments the site's link graph for those crawlers.

**Blocked resources** -- if robots.txt blocks CSS or JS files that the renderer needs, the page renders incorrectly or not at all.

**AJAX timeout behavior** -- content that loads from an API call after the initial render may not appear within Googlebot's rendering budget. Anything that requires user interaction (click, scroll, hover) to load is invisible to all crawlers.

**Hydration and CSR detection** -- does the main content exist in the initial HTML, or is the page a shell that fills itself with JavaScript? This is the difference between SSR/SSG (safe) and CSR (dangerous for SEO).

**ISR/SSG cache state** -- for frameworks like Next.js, verify that pages are served from cache (`x-nextjs-cache: HIT`) rather than regenerating on every request, which risks timeouts and inconsistent content.

**Dynamic rendering detection** -- some sites serve different content to Googlebot than to regular users. This is not cloaking if done correctly (Google explicitly permits dynamic rendering as a temporary solution), but it creates maintenance burden and can drift.

### Tool invocations

**Screaming Frog (primary):**

Configure SF with JavaScript rendering enabled: Configuration > Spider > Rendering > JavaScript. Set AJAX timeout to 5 seconds. Crawl the site, then compare:

```
Export: Internal > All (with rendered HTML stored)
# Compare the "Indexability" column between JS-rendered and raw crawl modes
# Any page indexable only with JS rendering is a risk
```

Check blocked resources:

```
Export: Internal > Blocked Resources
# Any CSS/JS file blocked by robots.txt that affects rendering
```

**Chrome DevTools (CSR detection):**

```javascript
// evaluate_script: Check if main content exists before JS executes
// This runs against the raw HTML, before hydration
document.querySelectorAll('main, article, [role="main"]').length > 0
  ? 'SSR/SSG: main content in initial HTML'
  : 'CSR WARNING: no main content element in initial HTML'
```

**Bash (rendered vs raw comparison + cache headers):**

```bash
# Raw HTML -- what non-rendering crawlers see
curl -sL https://example.com/ | wc -c

# Check ISR/SSG cache state
curl -sI https://example.com/ | grep -iE 'x-nextjs-cache|x-vercel-cache|cf-cache-status'

# Dynamic rendering detection: compare Googlebot vs Chrome response
curl -sL -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" https://example.com/ | wc -c
curl -sL -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" https://example.com/ | wc -c
# Significant size difference indicates dynamic rendering or cloaking
```

### What good looks like

- Raw HTML contains all primary content, headings, and internal links. The rendered DOM adds interactivity and styling but not content.
- `x-nextjs-cache: HIT` or `cf-cache-status: HIT` on content pages.
- Zero blocked JS/CSS resources in the SF export.
- Identical or near-identical content served to Googlebot and regular user agents.

### What bad looks like

- Raw HTML is a shell (`<div id="root"></div>` with no content). This is a pure CSR app and is a P0 for any page that needs to rank.
- `x-nextjs-cache: MISS` on every request, indicating ISR is misconfigured or the revalidation route is broken.
- Blocked resources report shows the site's own JS bundles blocked by robots.txt.
- Googlebot receives a substantially different page than Chrome (unintentional cloaking).

### Stack-specific notes

- **Next.js App Router (RSC):** `view-source` on an RSC page can look bare because React Server Components stream content. The rendered DOM is correct, but raw crawlers may see incomplete HTML. Verify with SF JS rendering that the content is present in the initial server response, not just the streamed update.
- **Framer and Webflow:** historically CSR-heavy. Framer in particular must be validated with a rendered DOM check -- Lighthouse's "Document content with JS disabled" audit catches this.
- **SPAs (custom React/Vue without SSR):** the rendered DOM gap is usually catastrophic. If the site is a pure SPA with no SSR or pre-rendering, flag as P0 and recommend pre-rendering or SSR before launch.

---

## 3. Crawlability and Error Handling

**Priority: P0-P1.**

Broken pages, redirect loops, orphaned content, and soft 404s all waste crawl budget and create dead ends for users and search engines alike. But the most dangerous issue in this category is the soft 404: a page that returns a 200 status code while displaying "page not found" content. Google may attempt to index it, the URL occupies a slot in your crawl budget, and you get zero value from the indexation.

### What to check

**Internal 4xx and 5xx errors** -- every broken internal link is a page that users and crawlers hit, follow, and find nothing. A 500 error on a critical page during Googlebot's crawl window can cause that page to drop from the index entirely.

**Redirect chains and loops** -- each redirect hop adds latency and costs crawl budget. Chains longer than 2 hops risk being abandoned by crawlers. Loops are infinite and will cause the crawler to give up on the URL permanently.

**Orphan pages** -- pages that exist (they return 200, they are in the sitemap or CMS) but have zero internal links pointing to them. Crawlers discover pages through links. An orphan page is a page that search engines may never find, or find only through the sitemap with reduced authority.

**Crawl depth** -- pages more than 4 clicks from the homepage receive exponentially less crawl frequency and pass less authority. For a pre-launch site, critical conversion or content pages buried at depth 5+ are a structural problem.

**Soft 404 detection** -- the silent killer. A page that returns HTTP 200 but displays "page not found," "no results found," or similar error content. Common causes: search results pages with no results, filtered category pages with zero products, dynamic routes that resolve to a template but have no data.

**Custom 404 page audit** -- the 404 page itself must return a proper 404 status code (not 200 or 301). It should also be useful: include navigation, search, popular pages. And it must work with JavaScript disabled, because crawlers hitting a 404 do not render JS.

### Tool invocations

**Screaming Frog (primary):**

```
Export: Response Codes > Response Codes:Client Error (4xx)
Export: Response Codes > Response Codes:Server Error (5xx)
Export: Reports > Redirect Chains
Export: Reports > Redirect & Canonical Chains
```

For orphan pages, run Crawl Analysis (post-crawl): Reports > Crawl Analysis > configure with sitemap.xml import. The "Orphan URLs" report surfaces pages in the sitemap but not reachable via internal links.

Check crawl depth: the "Crawl Depth" column in Internal > All. Filter for depth > 4.

**Soft 404 detection (SF Custom Search):**

```
Custom Search: (?i)(page not found|404|no results|doesn't exist)
Filter: Status Code = 200
# Any match is a soft 404 candidate
```

This custom search scans page text for error-content patterns on pages that return 200. Review each match manually -- some may be legitimate references to "404" in blog content, but most are genuine soft 404s.

**Bash (404 page audit):**

```bash
# Does the 404 page return a proper 404 status code?
curl -sI https://example.com/guaranteed-nonexistent-page-xyz123
# Expect: HTTP/2 404 (or HTTP/1.1 404)
# Bad: HTTP/2 200 (soft 404 at the server level)
# Bad: HTTP/2 301 (redirecting 404s to homepage -- wastes crawl budget, confuses signals)

# Does the 404 page have useful content (not a blank page or a JS-only render)?
curl -sL https://example.com/guaranteed-nonexistent-page-xyz123 | grep -c '<a '
# Should show navigation links. Zero links = unhelpful 404 page.
```

### What good looks like

- Zero internal links pointing to 4xx or 5xx pages.
- No redirect chains longer than 2 hops. No redirect loops.
- Orphan pages report is empty (every page in the sitemap is reachable via internal links).
- All priority pages at crawl depth 3 or less from the homepage.
- Custom search for soft 404 patterns returns zero matches on 200-status pages.
- The 404 page returns a 404 status code, includes navigation, and works without JavaScript.

### What bad looks like

- Internal links to 404 pages, especially in global navigation or footer.
- Redirect chains from old staging URLs through multiple hops to production.
- High-value pages orphaned -- in the sitemap but no internal links.
- Conversion pages at crawl depth 5+.
- Search results pages returning 200 with "no results found" content (Google will attempt to index these).
- 404 page returns 200 or redirects to the homepage.

### Consultant interpretation example

> "Your site has 23 URLs returning 200 that contain 'no results found' in the page body. These are search and filter pages with zero matching products. Google sees 23 indexable pages with no useful content -- it will either waste crawl budget attempting to re-crawl them, or classify them as soft 404s and question the quality of your site overall. The fix is to return a proper 404 status code when a search or filter produces zero results, or to render a useful fallback (popular products, related categories) and add `noindex` to prevent these empty states from occupying index slots."

---

## 4. Canonicalization

**Priority: P1 -- P0 if staging hostnames leak.**

Canonical tags tell search engines which version of a page is the "original" when multiple URLs serve similar or identical content. When canonicalization is broken, search engines must guess which URL to index, and they guess wrong often enough to cause real ranking problems. But the pre-launch catastrophe is simpler: staging hostnames in canonical tags.

If your canonical tags point to `https://staging.example.com/page` instead of `https://example.com/page`, Google will attempt to index the staging URL (if accessible) or ignore the production URL entirely (if staging is blocked). Either outcome is a full indexation failure for every affected page.

### What to check

- **Missing canonicals** -- pages without any canonical tag. Search engines infer one, but inference is unreliable, especially when query parameters, trailing slashes, and protocol variations create multiple URLs for the same content.
- **Non-indexable canonical targets** -- a canonical pointing to a page that is itself noindexed, 404, or redirected. This creates a logical contradiction that search engines resolve unpredictably.
- **Multiple conflicting canonicals** -- more than one canonical tag on a page, or a canonical tag that disagrees with the `X-Canonical-URL` header. Search engines pick one; you do not control which.
- **Staging hostname leaks** -- canonical tags, OG URLs, or schema `@id` values containing staging, dev, or localhost hostnames. The single most common pre-launch SEO mistake.
- **Trailing slash + canonical interaction** -- if `example.com/page` and `example.com/page/` both resolve (200), the canonical must pick one. If the canonical says `/page/` but the server redirects `/page/` to `/page`, there is a conflict.
- **Query parameter + canonical interaction** -- parameters like `?utm_source=`, `?ref=`, or `?fbclid=` create duplicate URLs. The canonical should point to the clean URL without parameters.

### Tool invocations

**Screaming Frog (primary):**

```
Export: Canonicals > Canonicals:Missing
Export: Canonicals > Canonicals:Non-Indexable Canonical
Export: Canonicals > Canonicals:Multiple Conflicting
```

Additionally, export Internal > All and filter the "Canonical Link Element 1" column for any value containing `staging`, `dev`, `localhost`, or any non-production hostname.

**Bash (spot-check):**

```bash
# Check canonical tag
curl -sL https://example.com/page | grep -i 'rel="canonical"'

# Check for staging hostname in canonical
curl -sL https://example.com/ | grep -i 'rel="canonical"' | grep -iE 'staging|dev\.|localhost|\.local'

# Trailing slash behavior
curl -sIL https://example.com/page
curl -sIL https://example.com/page/
# Both should either resolve to the same URL or one should 301 to the other
```

### What good looks like

- Every indexable page has exactly one self-referencing canonical tag pointing to the production URL.
- All canonical targets return 200 and are indexable.
- Trailing slash behavior is consistent: one form 301-redirects to the other, and the canonical matches the surviving form.
- Zero canonical tags containing staging, dev, or localhost hostnames.

### What bad looks like

- Canonical tags pointing to `https://staging.example.com/...`.
- Pages with two canonical tags (e.g., one from the CMS and one from a plugin).
- Canonical pointing to a 301 redirect target.
- Canonical on `example.com/page/` pointing to `example.com/page` while the server serves both as 200 with identical content.
- Query parameters like `?ref=twitter` creating canonical confusion.

---

## 5. Sitemaps

**Priority: P1.**

The XML sitemap is the site's self-declaration of which URLs matter and should be indexed. It supplements crawling -- it does not replace it, and it does not guarantee indexation. But a broken or inaccurate sitemap actively misleads search engines: it wastes crawl budget on URLs that should not be indexed and fails to surface URLs that should be.

### What to check

- **Existence and accessibility** -- `sitemap.xml` or `sitemap_index.xml` at the root, returning 200 with valid XML.
- **robots.txt reference** -- the sitemap URL should appear as `Sitemap: https://example.com/sitemap.xml` in robots.txt. Without this, search engines must discover it through GSC submission alone.
- **Content accuracy** -- every URL in the sitemap should be canonical, indexable, and return 200. No noindexed pages, no redirects, no 404s, no non-canonical URLs, and critically, no staging URLs.
- **Validation** -- valid XML per the sitemaps protocol. Common errors: missing namespace declaration, invalid `<lastmod>` date format, URLs exceeding 50,000 per sitemap file.

### Tool invocations

**Screaming Frog (primary):**

SF crawls linked sitemaps automatically when configured (Configuration > Spider > Crawl > Crawl Linked XML Sitemaps). After crawl:

```
Export: Sitemaps > Sitemap URLs
# Cross-reference against Indexability column
# Filter for any URL with Indexability != "Indexable"
```

**Bash (manual check):**

```bash
# Does the sitemap exist?
curl -sI https://example.com/sitemap.xml
# Expect: 200 with Content-Type: application/xml or text/xml

# Is it referenced in robots.txt?
curl -s https://example.com/robots.txt | grep -i sitemap

# Quick content check -- any staging URLs?
curl -s https://example.com/sitemap.xml | grep -iE 'staging|dev\.|localhost'

# URL count
curl -s https://example.com/sitemap.xml | grep -c '<loc>'
```

### What good looks like

- Sitemap exists, is valid XML, and is referenced in robots.txt.
- Every URL in the sitemap returns 200, is self-canonicalized, and is indexable.
- `<lastmod>` dates are accurate (not all set to the same date or the build date).
- URL count in the sitemap roughly matches the number of indexable pages found by crawling.

### What bad looks like

- No sitemap at all.
- Sitemap contains staging URLs (`https://staging.example.com/...`).
- Sitemap includes noindexed pages, 301 redirects, or 404s.
- Sitemap not referenced in robots.txt.
- `<lastmod>` dates are all identical or clearly wrong (e.g., year 2000).

---

## 6. Redirects

**Priority: P1.**

Redirect hygiene determines whether link equity from the old site (or from staging) transfers cleanly to production, whether users arriving via bookmarks or external links reach the right page, and whether crawl budget is spent on content rather than redirect chains.

### What to check

- **HTTP to HTTPS** -- all HTTP URLs must 301 to their HTTPS equivalents. No exceptions.
- **www consistency** -- either `www.example.com` redirects to `example.com` or vice versa, via 301. Both should not resolve to 200.
- **Trailing slash policy** -- pick one and redirect the other with 301. The choice does not matter; consistency does.
- **Redirect chains** -- any redirect sequence longer than 2 hops. Each hop adds latency and costs crawl budget. Chains of 5+ hops may be abandoned by crawlers.
- **301 vs 302 correctness** -- permanent moves (staging to production, old URL structure to new) must be 301. A 302 tells search engines the old URL might come back, so they continue to index it. Staging-to-production redirects that are 302 instead of 301 mean Google keeps trying to index the staging URL.
- **Mixed-case URL handling** -- `/About-Us` and `/about-us` should resolve to one canonical form via 301. Case sensitivity is server-dependent (Linux is case-sensitive, most CDNs are not by default).

### Tool invocations

**Screaming Frog (primary):**

```
Export: Reports > Redirect Chains
Export: Reports > Redirect & Canonical Chains
Export: Response Codes > Response Codes:Redirection (3xx)
# Filter the 3xx export by Status Code to separate 301 from 302/307/308
```

**Bash (spot-checks):**

```bash
# HTTP -> HTTPS redirect
curl -sI http://example.com/ | head -5
# Expect: 301 with Location: https://example.com/

# www redirect
curl -sI https://www.example.com/ | head -5
# Expect: 301 to https://example.com/ (or vice versa)

# Trailing slash behavior
curl -sI https://example.com/about | head -5
curl -sI https://example.com/about/ | head -5
# One should 301 to the other, not both return 200

# Check for redirect chains
curl -sIL https://example.com/old-page 2>&1 | grep -E 'HTTP/|Location:'
# Count the hops
```

### What good looks like

- HTTP 301 redirects to HTTPS on every URL.
- www/non-www is consistent with 301 enforcement.
- Trailing slash policy is consistent sitewide.
- Zero redirect chains longer than 2 hops.
- All permanent redirects use 301 (or 308); 302 is used only for genuinely temporary redirects.

### What bad looks like

- HTTP and HTTPS both serve 200 (no redirect at all -- duplicate content).
- www and non-www both serve 200 (duplicate content).
- Redirect chains: `http://www.example.com` -> `https://www.example.com` -> `https://example.com` -> `https://example.com/` (4 hops for what should be 1).
- Staging-to-production redirects using 302 instead of 301.

---

## 7. Structured Data

**Priority: P1-P2.**

Structured data (JSON-LD) helps search engines understand what a page is about and can unlock rich results (review stars, product prices, event dates, breadcrumbs) that dramatically improve click-through rates. In 2026, it also serves as a citation signal for AI search engines: pages with valid Organization + Article + Person entities are disproportionately cited in AI-generated answers.

But structured data has a shelf life. Templates built in 2022 may contain schema types that Google has since deprecated, and implementing deprecated schema is not neutral -- it signals that the site is not maintained.

### What to check

**Validation** -- every page with structured data should pass both syntax validation (valid JSON-LD, correct `@context`) and eligibility validation (the right properties for the `@type`, meeting Google's specific requirements for rich results).

**Correct @type for page type** -- an article page should have `Article` or `NewsArticle`, not `WebPage`. A product page should have `Product` with `offers`, not just `Product` without pricing.

**2026 reality check -- deprecated schema:**
- **FAQ rich results** were restricted in August 2023 to authoritative health and government sites only. If your site is not a government health agency, FAQ schema will be silently ignored. Do not implement it on commercial pages.
- **HowTo rich results** were fully deprecated and removed from search results on September 13, 2023. If HowTo schema is still in your templates, remove it. It does nothing, and its presence signals stale templates.

**AI citation signals:**
- **Organization schema** with `name`, `url`, `logo`, `sameAs` (linking to Wikipedia, LinkedIn, Crunchbase, social profiles). This establishes entity identity for AI systems.
- **Article schema** with `author` linking to a `Person` entity with `sameAs` to the author's external profiles. AI search engines use author identity as a credibility signal.
- **`max-image-preview:large`** in the robots meta tag enables images to appear in Google Discover and AI Overviews. Without it, your content is eligible for text citations but not visual ones.

### Tool invocations

**Screaming Frog (primary):**

Enable JSON-LD extraction: Configuration > Spider > Extraction > JSON-LD. After crawl:

```
Export: Structured Data > Structured Data:Validation Errors
Export: Structured Data > Structured Data:Validation Warnings
```

Custom extraction for JSON-LD @type (see SF Power Workflow section at the end of this document):

```
XPath: //script[@type='application/ld+json']
Mode: Extract Inner HTML
```

**Rich Results Test (manual, per-page):**

```
https://search.google.com/test/rich-results?url=https://example.com/page
```

**Bash (spot-check):**

```bash
# Extract JSON-LD blocks
curl -sL https://example.com/ | grep -oP '<script type="application/ld\+json">.*?</script>'

# Check for deprecated schema types
curl -sL https://example.com/ | grep -oP '"@type"\s*:\s*"[^"]*"' | sort -u
# Flag: FAQPage (unless health/gov), HowTo (always remove)

# Check max-image-preview
curl -sL https://example.com/ | grep -i 'max-image-preview'
```

### What good looks like

- Every page has at least one JSON-LD block with valid syntax and the correct `@type` for its content.
- Organization schema on the homepage with `sameAs` to external profiles.
- Article/NewsArticle schema on content pages with `author` linking to a Person entity.
- `max-image-preview:large` in the robots meta tag.
- Zero validation errors in SF or Rich Results Test.
- No FAQPage or HowTo schema on commercial pages.

### What bad looks like

- No structured data at all.
- FAQPage schema on every page of a commercial site (Google ignores it since August 2023).
- HowTo schema still in templates (fully deprecated September 13, 2023).
- Article schema missing `author` or `datePublished`.
- Organization schema missing `sameAs`.
- JSON-LD with syntax errors (missing `@context`, trailing commas, unescaped characters).
- `max-image-preview` not set or set to `standard` (limits AI Overviews and Discover eligibility).

### Consultant interpretation example

> "Your article pages use valid Article schema with proper datePublished and dateModified -- that's good. However, three things need attention. First, none of your articles have an author Person entity with sameAs links. AI search engines like Perplexity and ChatGPT Search use author identity as a citation signal; without it, your content has no author-level authority signal. Second, your site still has HowTo schema on 12 how-to guide pages. HowTo rich results were fully removed from Google Search on September 13, 2023. The schema is dead weight -- remove it from your templates. Third, you're missing `max-image-preview:large` in your robots meta tag. Without it, your hero images cannot appear in Google Discover or AI Overviews, limiting your visual search presence."

---

## 8. Hreflang (If International)

**Priority: P1 for international sites, skip otherwise.**

Hreflang tells search engines which language and regional version of a page to show to which users. When it breaks, French users see the English page, German users see the Spanish page, and all versions compete against each other in the same SERP -- cannibalizing each other instead of dominating their respective markets.

### What to check

- **Reciprocal links** -- if page A declares `hreflang="fr"` pointing to page B, then page B must declare `hreflang="en"` pointing back to page A. Non-reciprocal hreflang is ignored by Google.
- **Non-200 targets** -- every URL referenced in an hreflang annotation must return 200. A hreflang pointing to a 301, 404, or noindexed page is invalid.
- **ISO code correctness** -- language codes must follow ISO 639-1 (2-letter) and country codes must follow ISO 3166-1 alpha-2. The most common mistake: `en-UK` instead of `en-GB`. There is no `UK` in ISO 3166.
- **x-default** -- the fallback declaration for users whose language/region does not match any hreflang annotation. Required for proper international targeting.

### Tool invocations

**Screaming Frog (primary):**

SF has a dedicated Hreflang tab that validates reciprocity, target status codes, and ISO code correctness automatically.

```
Export: Hreflang > Hreflang (all reports)
# Check for: Missing Return Links, Non-200 Hreflang, Inconsistent Language
```

**Bash (spot-check):**

```bash
# Extract hreflang annotations
curl -sL https://example.com/ | grep -i 'hreflang'

# Verify reciprocity manually
curl -sL https://example.com/fr/ | grep -i 'hreflang.*en'
# Should find a link back to the English version
```

### What good looks like

- Every page in every language version has complete, reciprocal hreflang annotations.
- All hreflang targets return 200 and are indexable.
- ISO codes are correct (`en-GB`, not `en-UK`; `pt-BR`, not `pt-BRA`).
- `x-default` is declared and points to the most broadly applicable version.

### What bad looks like

- Hreflang annotations on the English version but not on the French version (non-reciprocal).
- `hreflang="en-UK"` (invalid ISO code).
- Hreflang pointing to staging URLs or redirected pages.
- Missing `x-default`.

---

## 9. Staging Leak Sweep

**Priority: P0 if found, P2 as routine check.**

Staging leaks are references to staging, development, or local hostnames that survived the production deployment. They appear in canonical tags, Open Graph tags, JSON-LD schema, sitemap URLs, and hardcoded absolute internal links. Each one is a vector for indexation failure (canonicals), social sharing failure (OG tags), or entity confusion (schema).

This check overlaps with Canonicalization (check 4) but is broader -- it sweeps every location where a hostname appears, not just the canonical tag.

### What to check

Scan the entire crawled HTML for staging hostname patterns. Common patterns: `staging.`, `dev.`, `.local`, `localhost`, `preview.`, `test.`, internal IP addresses.

Specific locations to verify:
- Canonical tags (`<link rel="canonical">`)
- Open Graph tags (`og:url`, `og:image`)
- JSON-LD schema (`@id`, `url`, `image`, `mainEntityOfPage`)
- Sitemap URLs
- Absolute internal links (`<a href="https://staging...">`)
- Asset URLs (images, scripts loading from staging CDN)

### Tool invocations

**Screaming Frog (Custom Search):**

```
Custom Search: (?i)(staging\.|dev\.|test\.|\.local|localhost|127\.0\.0\.1|192\.168\.)
Type: Contains
Search In: HTML (covers all tags, schema, links, assets)
```

**Bash (targeted checks):**

```bash
# Check canonical for staging
curl -sL https://example.com/ | grep -i 'rel="canonical"' | grep -iE 'staging|dev\.|localhost'

# Check OG tags for staging
curl -sL https://example.com/ | grep -i 'og:' | grep -iE 'staging|dev\.|localhost'

# Check JSON-LD for staging
curl -sL https://example.com/ | grep -oP '<script type="application/ld\+json">.*?</script>' | grep -iE 'staging|dev\.|localhost'

# Check sitemap for staging
curl -s https://example.com/sitemap.xml | grep -iE 'staging|dev\.|localhost'
```

### What good looks like

- Zero references to staging, dev, test, local, or localhost hostnames anywhere in the production site.

### What bad looks like

- Canonical tag pointing to `https://staging.example.com/page`.
- `og:image` loading from `https://dev.example.com/images/hero.jpg`.
- JSON-LD `@id` set to `https://localhost:3000/page`.
- Sitemap containing `https://staging.example.com/` URLs.
- Internal links hardcoded to staging domain.

---

## 10. Pagination and Faceted Navigation

**Priority: P2 for most sites, P1 for e-commerce or content-heavy sites.**

Pagination and faceted navigation create URL proliferation. A blog with 500 posts and 10-per-page pagination creates 50 paginated listing pages. An e-commerce site with 3 color options, 5 sizes, and 4 sort orders on each of 200 category pages creates 200 x 60 = 12,000 faceted URLs, most of which are thin duplicates of each other. Uncontrolled, this URL explosion wastes crawl budget, dilutes link equity, and can trigger Google's duplicate content filters.

### What to check

**Pagination:**

- **rel=next/prev is deprecated.** Google announced in 2019 that it no longer uses `rel="next"` and `rel="prev"` for pagination signals. Do not implement it on a new site. Do not spend time adding it to an existing site.
- **Self-referencing canonicals** -- each paginated page (`/blog/page/2`, `/blog/page/3`) should have a self-referencing canonical pointing to itself, not a canonical pointing back to page 1. Google explicitly warned against canonicalizing all pages to page 1, as this tells Google to only index page 1 and ignore all content on subsequent pages.
- **Indexability** -- paginated pages should be indexable (no `noindex`) unless they serve no ranking purpose. If you noindex paginated listing pages, the articles on pages 2+ lose their primary discovery path.
- **Orphaned content** -- articles that only appear on page 5+ of a paginated listing are effectively orphaned unless they also have direct internal links from other content.

**Faceted navigation:**

- **Parameter handling in robots.txt** -- filter parameters (color, size, sort, brand) should be blocked via `Disallow` rules to prevent crawlers from attempting to crawl every combination.
- **Canonical strategy** -- filtered pages should canonical back to the base category page. The filter version is a view of the same content, not a distinct page.
- **noindex,follow on low-value combinations** -- for filter combinations that are too thin to rank but contain links to products, use `noindex,follow`. This keeps link equity flowing while preventing index bloat.
- **Never nofollow internal facet links** -- if you want a faceted page crawled (for link discovery) but not indexed, use `noindex,follow`, not `nofollow` on the link itself. Nofollow on internal links wastes the link equity entirely.
- **Query string limits** -- SF can be configured to limit the number of query parameters it follows (Configuration > Spider > Limits). Without this, an SF crawl of a large e-commerce site can attempt millions of faceted URLs and never complete.

### Tool invocations

**Screaming Frog (primary):**

```
# Identify parameterized URL volume
Export: Internal > All
Filter: URL > Contains > ?
# Count of URLs with query parameters indicates faceted navigation scale

# Custom search for common facet patterns
Custom Search: (?i)(\?|&)(color|size|sort|filter|page|p|brand|category|price)=
Type: Contains
Search In: URL
```

For pagination, check the canonical column on paginated URLs to confirm self-referencing canonicals.

**Bash (spot-check):**

```bash
# Check a paginated page's canonical
curl -sL https://example.com/blog/page/2 | grep -i 'rel="canonical"'
# Should point to itself: /blog/page/2, NOT to /blog or /blog/page/1

# Check faceted URLs
curl -sL "https://example.com/category?color=red&size=large" | grep -i 'rel="canonical"'
# Should canonical to the base category: /category

# Check robots.txt for parameter handling
curl -s https://example.com/robots.txt | grep -E 'Disallow.*\?|Disallow.*(color|size|sort|filter)'
```

### Stack-specific notes

- **Shopify:** the `?variant=` parameter creates near-infinite faceted URLs for every product. Shopify generates a separate URL for every variant combination. These should be handled with canonical tags (which Shopify does by default for variant pages, but verify).
- **WooCommerce:** `?pa_` attributes (product attributes like `?pa_color=red`) create faceted URLs. Unlike Shopify, WooCommerce does not automatically canonical these -- the theme or an SEO plugin must handle it.
- **Custom SPAs:** often lack any facet URL strategy entirely. Filters may not create distinct URLs at all (state is in memory only), which means filtered views are not crawlable. This is either fine (if the base category page is sufficient) or a problem (if filtered views have unique ranking potential).

### What good looks like

- Paginated pages have self-referencing canonicals.
- No `rel="next"` / `rel="prev"` on a new site (deprecated 2019).
- Faceted URLs canonical to the base category page.
- robots.txt blocks crawling of filter parameters.
- Low-value filter combinations are `noindex,follow`.
- SF crawl completes in a reasonable time without exploding into millions of parameterized URLs.

### What bad looks like

- All paginated pages canonical to page 1 (Google warned against this).
- Thousands of faceted URLs with no canonical, no noindex, and no robots.txt coverage.
- `nofollow` on internal links to faceted pages (wastes link equity).
- SF crawl never completes because it follows every parameter combination to infinity.
- Shopify `?variant=` URLs without canonical handling.

---

## SF Power Workflow: Custom Extractions and Searches

This section consolidates the Screaming Frog custom configurations that make the 10 checks above more efficient when run together as a pre-launch audit. Configure these before starting the crawl so all data is captured in a single pass.

### Custom Extractions

Set up under Configuration > Custom > Extraction. These add columns to the SF export for every crawled URL.

| What | XPath / CSS | Mode | Why |
|---|---|---|---|
| JSON-LD @type | `//script[@type='application/ld+json']` | Extract Inner HTML | Parse the JSON to identify schema types per page. Enables bulk structured data audit without per-page Rich Results Test. |
| OG image | `//meta[@property='og:image']/@content` | Extract Text / Attribute: content | Surfaces staging hostnames in OG images and identifies pages missing OG images. |
| Publish date | `//meta[@property='article:published_time']/@content` | Extract Text / Attribute: content | Identifies stale or missing publication dates. AI search engines weight freshness. |
| Canonical | `//link[@rel='canonical']/@href` | Extract Text / Attribute: href | Bulk canonical audit in a single column. Faster than navigating SF's built-in canonical reports for cross-referencing. |
| H1 count | `count(//h1)` | Function Value | Surfaces pages with zero H1s (missing) or multiple H1s (usually a template error). Numeric output enables filtering. |

For the H1 count extraction, use XPath Function Value mode, which returns a number rather than text content. This allows filtering for values != 1 in the export.

### Custom Searches

Set up under Configuration > Custom > Search. These flag specific content patterns across all crawled pages.

| Purpose | Regex | Search In | Type | What it catches |
|---|---|---|---|---|
| Staging hostname | `(?i)(staging\.|dev\.|test\.|\.local\|localhost)` | HTML | Contains | Staging references in any HTML element: links, canonicals, schema, OG tags, asset URLs. |
| Lorem ipsum | `(?i)lorem ipsum` | Page Text | Contains | Placeholder content left in production templates. |
| Hardcoded HTTP | `(src\|href)=["']http://` | HTML | Contains | Mixed content -- assets or links using HTTP instead of HTTPS. Can trigger browser security warnings and block rendering. |
| Soft 404 content | `(?i)(page not found\|404\|no results\|doesn't exist)` | Page Text | Contains | Soft 404 pages returning 200 status with error content. Filter results by Status Code = 200 for true positives. |
| Facet parameters | `(?i)(\?\|&)(color\|size\|sort\|filter\|page\|p)=` | URL | Contains | Identifies the scale of faceted navigation URLs in the crawl. High counts indicate uncontrolled faceted nav. |
| Missing analytics | `gtag\|G-[A-Z0-9]+` | HTML | Does Not Contain | Pages where Google Analytics 4 is not firing. Could indicate template errors or conditional loading issues. |
| Console.log/debug | `console\.(log\|error\|warn)` | Rendered HTML | Contains | Debug statements left in production JavaScript. Not an SEO issue directly, but a code quality signal and potential information leak. |

### Crawl Configuration for Pre-Launch

When running the full technical SEO audit, configure SF before starting the crawl:

- **Storage mode:** Database (Configuration > System > Storage) -- prevents memory exhaustion on large crawls.
- **Rendering:** JavaScript enabled (Configuration > Spider > Rendering) for JS-dependent stacks. Use Googlebot Smartphone user agent.
- **Disable respect directives:** Turn off "Respect Noindex," "Respect Canonicals," and "Respect Robots.txt" in audit mode. You want to see the directives, not obey them. SF reports what directives exist regardless, but disabling respect ensures you crawl every URL including blocked ones.
- **Store HTML:** Enable both raw and rendered HTML storage for later comparison.
- **Crawl sitemaps:** Enable "Crawl Linked XML Sitemaps" to discover and validate sitemap content.
- **AJAX timeout:** 5 seconds (Configuration > Spider > Rendering > AJAX Timeout) to match Googlebot's rendering budget.
- **Rate limit:** 5 concurrent threads maximum on staging environments to avoid overloading development servers.
- **Crawl Analysis:** Run post-crawl (Reports > Crawl Analysis) with sitemap import for orphan page detection.
