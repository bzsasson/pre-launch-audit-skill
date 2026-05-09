# Stack Profiles: Fingerprinting, Recon, and Platform-Specific Audit Guidance

This reference is the stack detection and adaptation layer for the pre-launch website audit skill. Every sub-audit -- technical SEO, AI accessibility, security, performance, on-page -- changes shape depending on the framework and hosting platform underneath. Detect the stack first, then branch.

The fingerprint table, bash commands, and quirk profiles below are verified against 2025-2026 production deployments. Wappalyzer was archived in 2023; do not depend on it as a runtime tool. The bash recon layers here, supplemented by DataForSEO technology detection, are the primary identification method.

---

## 1. Fingerprint Cheat Sheet

Fifteen stacks, three signal categories. A match in any column is a signal; two or more columns is high confidence.

| Stack | Header Signal | HTML / Path Signal | DNS / Infra |
|---|---|---|---|
| **Next.js** | `x-powered-by: Next.js`, `x-nextjs-cache: HIT/MISS/STALE`, `x-vercel-id`, `x-vercel-cache` | `__NEXT_DATA__` JSON script tag, `/_next/static/`, `/_next/image?url=` | CNAME `cname.vercel-dns.com` |
| **Nuxt** | `x-powered-by: Nuxt` | `window.__NUXT__`, `/_nuxt/` | -- |
| **Astro** | -- | `/_astro/`, `data-astro-cid-*` attributes, `<meta name="generator" content="Astro">` | -- |
| **SvelteKit** | -- | `__sveltekit_*` globals, `/_app/immutable/` | Often Vercel or Cloudflare |
| **Remix** | -- | `__remixContext`, `/build/`, `data-remix-*` attributes | -- |
| **Gatsby** | -- | `___gatsby` div, `/page-data/` JSON files, `gatsby-*-css` classes | Often Netlify |
| **WordPress** | `x-pingback`, `link: <...>; rel="https://api.w.org/"` (wp-json) | `/wp-content/`, `/wp-includes/`, `<meta name="generator" content="WordPress ...">`, `/?p=` query strings | -- |
| **Shopify** | `x-shopid`, `x-shopify-stage`, `x-sorting-hat-shopid` | `cdn.shopify.com`, `Shopify.theme`, `window.Shopify` object | CNAME `shops.myshopify.com` |
| **Webflow** | `x-wf-page-id`, `server: Webflow` | `data-wf-page`, `data-wf-site`, `webflow.js` script | CNAME `proxy-ssl.webflow.com` |
| **Framer** | -- | `framerusercontent.com` asset URLs, `data-framer-*` attributes | CNAME `framer.website` |
| **Wix** | `x-wix-request-id`, `server: Pepyaka` | `wixstatic.com` asset URLs, `wix-*` CSS classes | -- |
| **Squarespace** | `server: Squarespace` | `static1.squarespace.com`, `Static.SQUARESPACE_CONTEXT` global | -- |
| **Hugo / Jekyll / Eleventy** | Plain `nginx` or CDN headers (no framework signature) | `<meta name="generator" content="Hugo">` (or Jekyll, or Eleventy) | Often Netlify or Cloudflare Pages |
| **Drupal** | `x-drupal-cache`, `x-drupal-dynamic-cache`, `x-generator: Drupal ...` | `/sites/default/files/`, `Drupal.settings` global | -- |
| **CDN / Edge** | `cf-ray` (Cloudflare), `x-vercel-id` (Vercel), `x-served-by` (Fastly), `x-amz-cf-id` (CloudFront), `x-nf-request-id` (Netlify) | -- | -- |

CDN detection is not mutually exclusive with framework detection. A Next.js site on Vercel behind Cloudflare will produce signals in all three rows. Record each layer separately.

---

## 2. Bash Recon Commands (5 Layers)

Run these in order. Each layer adds context to the previous one. All commands are non-destructive and safe for production sites.

### Layer 1 -- Headers and Status

Three requests with different user agents. The first is baseline. The second detects cloaking (serving different content to search engines). The third checks whether the site blocks AI crawlers at the HTTP level.

```bash
# Baseline -- follow redirects, capture all hop headers
curl -sIL -A "Mozilla/5.0" https://example.com

# Cloaking check -- does the site serve different headers/status to Googlebot?
curl -sI -H "User-Agent: Googlebot/2.1 (+http://www.google.com/bot.html)" https://example.com

# AI-bot delivery check -- 403 or challenge page means invisible to ChatGPT Search
curl -sI -H "User-Agent: GPTBot/1.0 (+https://openai.com/gptbot)" https://example.com
```

Compare status codes and key headers across all three. A 403 on the GPTBot request when the others return 200 means Cloudflare Bot Fight Mode, Wordfence, or an explicit robots-enforcement WAF rule is active.

### Layer 2 -- Root Files

These files reveal platform defaults, sitemap structure, and whether the site has prepared for AI crawlers or security researchers.

```bash
curl -s https://example.com/robots.txt
curl -s https://example.com/sitemap.xml | head -50
curl -s https://example.com/sitemap_index.xml | head -50
curl -s https://example.com/llms.txt
curl -s https://example.com/.well-known/security.txt
curl -s https://example.com/humans.txt
curl -sI https://example.com/favicon.ico
```

WordPress sites typically have auto-generated sitemaps at `/wp-sitemap.xml`. Shopify generates `/sitemap.xml` automatically. A missing `robots.txt` is itself a finding. An `llms.txt` presence on a non-documentation site is noted but not weighted -- it has no proven citation impact (ALLMO January 2026 study, SE Ranking 300k-domain study).

### Layer 3 -- HTML Signatures

Fetch the rendered homepage and scan for framework fingerprints. This catches stacks that do not advertise themselves via headers.

```bash
curl -sL https://example.com -o page.html
grep -Eo '__NEXT_DATA__|_nuxt|_astro|data-astro-cid|wp-content|wp-includes|cdn\.shopify|Shopify\.theme|webflow\.js|data-wf-page|framerusercontent|data-framer|squarespace|wixstatic|___gatsby|__sveltekit|__remixContext|Drupal\.settings|hugo|jekyll|eleventy' page.html | sort -u
```

For SPAs and CSR-heavy sites, the curl output may be a near-empty shell. That itself is a critical finding for the technical SEO sub-audit. If the grep returns nothing meaningful, the site likely renders client-side and needs a browser-based check (Chrome DevTools or Playwright) to see the actual DOM.

### Layer 4 -- DNS and TLS

DNS records reveal hosting platform, email infrastructure, and certificate provenance.

```bash
# Hosting platform via CNAME
dig +short example.com CNAME
dig +short www.example.com CNAME

# SPF, DKIM, DMARC for email security sub-audit
dig +short example.com TXT
dig +short example.com MX
dig +short _dmarc.example.com TXT

# Certificate chain and expiration
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates
```

A CNAME to `cname.vercel-dns.com` confirms Vercel. `proxy-ssl.webflow.com` confirms Webflow. `shops.myshopify.com` confirms Shopify. `framer.website` confirms Framer. No CNAME with an A record pointing to Cloudflare IPs (104.x.x.x or 172.x.x.x ranges) suggests Cloudflare proxying, which has major implications for AI crawler access.

### Layer 5 -- Bundle Inspection

Extract JavaScript source paths from the page. Framework-specific path patterns confirm the stack and reveal potential secrets exposure.

```bash
curl -sL https://example.com | grep -Eo 'src="[^"]+\.js"' | sort -u
```

Next.js bundles live under `/_next/static/`. Nuxt under `/_nuxt/`. Astro under `/_astro/`. Gatsby under `/page-data/`. WordPress plugin scripts under `/wp-content/plugins/`. Shopify app scripts under `cdn.shopify.com/extensions/`. Source map references (`.js.map`) in production are a security finding.

---

## 3. Stack-Specific Quirks

Each stack introduces audit considerations that a generic checklist misses. These are the most common traps, organized by framework.

### Next.js / Vercel

**ISR cache staleness.** When `x-nextjs-cache` returns `STALE`, the page is being served from a stale cache while revalidation happens in the background. A fix deployed today may not be visible to crawlers until on-demand revalidation runs. Confirm that `/api/revalidate` or equivalent `revalidatePath`/`revalidateTag` calls are wired for critical pages. During the audit, if you fix something and re-check immediately, you may see the old version -- this is expected ISR behavior, not a failed fix.

**App Router and React Server Components.** With RSC, `view-source` can be nearly bare -- the content is streamed as a binary RSC payload, not traditional HTML. The rendered DOM (what Chrome DevTools shows after hydration) is what matters for SEO. Always diff `curl` output against rendered DOM. If the curl output lacks the page's substantive content, that content depends on client-side RSC hydration, which training crawlers (GPTBot, ClaudeBot) historically cannot execute.

**`NEXT_PUBLIC_*` environment variables.** Any variable prefixed with `NEXT_PUBLIC_` is bundled into the client JavaScript by design. This is not a bug -- it is how Next.js exposes configuration to the browser. The audit must verify that no server-side secrets (API keys with write access, database credentials, service role tokens) use this prefix. Escape.tech's October 2025 study found 1 in 4 Vercel-hosted vibe-coded apps leaked server secrets through `NEXT_PUBLIC_*`.

**Server Actions.** App Router server actions are exposed as HTTP POST endpoints. Verify they have authentication middleware -- the framework does not add it by default.

### WordPress

**Plugin sprawl is the dominant risk surface.** The framework itself is reasonably hardened after two decades. The plugins are not. Every active plugin is an attack surface, a performance liability, and a potential source of injected JavaScript. The audit should inventory plugins via `/wp-json/wp/v2/plugins` (if exposed) or by scanning `/wp-content/plugins/` paths in the HTML.

**Username enumeration.** `/wp-json/wp/v2/users` returns usernames by default. Combined with `xmlrpc.php` (brute-force amplification) or `wp-login.php`, this is a credential-stuffing vector. Both should be restricted or disabled.

**AI bot blocking via security plugins.** Wordfence and iThemes Security can silently 403 GPTBot, ClaudeBot, and other AI crawlers without any indication in `robots.txt`. The only way to detect this is the Layer 1 GPTBot curl check. If the site owner wants AI search visibility, the WAF plugin's bot rules must be audited.

### Shopify

**Liquid renders server-side.** Shopify's templating engine produces clean HTML before it reaches the browser, which makes SEO generally straightforward. The rendered-DOM gap that plagues SPAs is not an issue here.

**Catalog data exposure.** `/products.json` and `/collections/all/products.json` expose the full product catalog as JSON. This is by design and generally harmless, but it exposes pricing, inventory status, and variant data to competitors. Flag it; do not recommend blocking it (Shopify does not support that).

**Infinite faceted URLs.** The `?variant=` parameter creates a unique URL for every product variant. Combined with collection filters, this can generate thousands of thin, near-duplicate URLs. Shopify's built-in canonical tags handle most cases, but custom theme modifications can break this. Verify that variant URLs canonical to the base product URL.

**App-injected scripts.** Shopify apps routinely inject JavaScript into the storefront. These scripts are the primary cause of poor Lighthouse performance scores on Shopify sites. Inventory them via Layer 5 bundle inspection and flag any that are render-blocking.

### Webflow and Framer

**CSR risk.** Framer is particularly CSR-heavy. Content rendered entirely in the browser may be invisible to crawlers that do not execute JavaScript. Run the rendered-DOM diff on every Framer site. Webflow is better but still uses client-side rendering for interactions and animations that can obscure content from raw HTML inspection.

**Webflow's Cloudflare migration.** Webflow is migrating all sites to Cloudflare infrastructure with a deadline of June 1, 2026. This migration enables Bot Fight Mode by default, which silently blocks AI crawlers. Any Webflow site audited after April 2026 must be checked for this. The site owner may need to configure Cloudflare's AI Crawl Control settings via Webflow's partnership dashboard or Cloudflare O2O (Orange-to-Orange) integration.

**Limited redirect control.** Both platforms offer redirect managers, but they lack regex support and bulk operations. Complex redirect strategies (trailing-slash normalization, pattern-based rewrites) may require a Cloudflare Worker or edge function in front of Webflow/Framer.

### Wix and Squarespace

**Platform ceilings.** These platforms impose hard limits that no audit recommendation can fix. The audit must flag these explicitly to avoid recommending the unfixable.

See Section 4 (Platform Ceilings) for the full list.

### SPAs (React, Vue, Angular without SSR)

**The rendered-DOM gap is usually catastrophic.** A single-page application that relies entirely on client-side rendering will show an empty or near-empty page to `curl`, to Googlebot's initial fetch (before rendering queue), and to every AI training crawler. This is a P0 finding. The fix is SSR, static pre-rendering, or a hybrid framework (Next.js, Nuxt, Analog). Dynamic rendering (serving pre-rendered HTML to bots) is a legacy stopgap that Google tolerates but does not recommend.

**Mandatory pre-render check.** For any SPA, compare `curl -sL` output against Chrome DevTools rendered DOM. If the title, headings, and body content are absent from the curl output, the site is invisible to non-rendering crawlers.

### Static Site Generators (Hugo, Jekyll, Eleventy)

**Trivially auditable.** Static HTML is the easiest stack to audit. There is no rendering gap, no server-side complexity, and no framework-specific security surface. The HTML is what it is.

**Brittle on dynamic features.** Forms, search, and 404 pages are the weak points. Static sites typically delegate these to third-party services (Netlify Forms, Algolia, Pagefind) or serverless functions. Verify that the 404 page returns a proper 404 status code (not 200), that form submissions work, and that search (if present) does not expose an unprotected API endpoint.

---

## 4. Platform Ceilings

Some platforms restrict what the site owner can control. The audit must not recommend fixes that the platform makes impossible. Flag these as platform limitations, not site owner failures.

**Wix:**
- Cannot customize HTTP security headers (no CSP, no Permissions-Policy, no COOP/CORP)
- Limited redirect control (no regex, no bulk, max 900 redirects)
- Restricted structured data (Wix auto-generates basic schema; manual JSON-LD injection requires workarounds)
- Cannot modify `robots.txt` granularly for individual AI bot user agents
- No server-side access for custom caching headers
- Cannot add Subresource Integrity (SRI) to third-party scripts

**Squarespace:**
- Same header limitations as Wix -- no custom security headers
- Limited redirect control (301 only, no 302/307/308, no regex)
- Structured data is template-driven; custom JSON-LD requires code injection
- No control over `X-Robots-Tag` headers
- Cannot serve different content or headers per user agent
- Cookie consent banner is platform-controlled with limited customization

**Shopify:**
- Better than Wix/Squarespace but still constrained
- Limited header control (can set some headers via `content_for_header` in Liquid, but no full CSP)
- No `.htaccess` or server config -- redirects via admin panel or `theme.liquid` URL redirect
- Automatic canonical tags work well but cannot be overridden per-page without app or custom code
- Cannot block specific crawler user agents at the server level (only via `robots.txt`)
- Rate-limited storefront API (separate from admin API) -- not a ceiling, but a factor in performance audits

When the audit encounters these platforms, the report should state: "This finding cannot be resolved within [Platform]. If [specific security/SEO requirement] is a business requirement, it requires migrating to a platform with server-level control, or placing a reverse proxy (Cloudflare Worker, Vercel Edge Middleware) in front of the platform."

---

## 5. Vibe-Coded Platforms

Lovable, Bolt, Base44, and Replit represent a distinct security archetype. These AI-assisted development platforms generate full-stack applications from natural language prompts. The code works. The security posture is frequently catastrophic.

Wiz, Escape.tech, and Autonoma published studies across 2025-2026 finding that approximately 20% of apps built on these platforms had critical vulnerabilities. CVE-2025-48757 (Lovable/Supabase RLS bypass, CVSS 9.3) and the Moltbook breach (1.5M tokens, 35k emails exposed) are the canonical case studies.

The following checks are mandatory on any site identified as vibe-coded.

### Supabase RLS Probe

Most vibe-coded apps use Supabase as their backend. The anon key is embedded in the client JavaScript by design -- this is expected and not itself a vulnerability. The vulnerability is when Row Level Security (RLS) policies are missing or misconfigured, allowing the anon key to read or write data it should not access.

**Anon vs. service_role key.** The anon key has limited privileges governed by RLS. The service_role key bypasses RLS entirely. If the service_role key appears in client-side JavaScript, that is a catastrophic finding -- full database access with no restrictions. The service_role key format is the same JWT structure as the anon key but decodes to `"role": "service_role"` instead of `"role": "anon"`. Check by base64-decoding the JWT payload segment.

**Probe procedure:** Extract the Supabase URL and anon key from the client bundle (search for `supabase.co` and the JWT). Then:

```bash
# List accessible tables via PostgREST
curl -s "$SUPABASE_URL/rest/v1/" \
  -H "apikey: $ANON_KEY" \
  -H "Authorization: Bearer $ANON_KEY" | jq .

# Probe each table for unauthorized read access
for table in users profiles orders payments settings; do
  echo "--- $table ---"
  curl -s "$SUPABASE_URL/rest/v1/$table?limit=1" \
    -H "apikey: $ANON_KEY" \
    -H "Authorization: Bearer $ANON_KEY"
done
```

Any 200 response with row data on a table that should be private (users, orders, payments, settings) means RLS is missing or broken.

### GraphQL Playground Exposure

Vibe-coded apps frequently expose a GraphQL playground at `/graphql` or `/graphiql`. This is an interactive query interface that lets anyone browse the full schema and execute arbitrary queries. In production, this must be disabled or authenticated.

```bash
curl -s https://example.com/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name } } }"}'
curl -s https://example.com/graphiql
```

A successful introspection response or a rendered playground page is a finding.

### AI Endpoint Rate Limiting

Apps with AI features (chatbots, content generators, summarizers) expose endpoints like `/api/chat`, `/api/generate`, `/api/completions`. These endpoints proxy requests to OpenAI, Anthropic, or other providers using the app owner's API key. Without rate limiting, anyone can drain the owner's API budget.

```bash
# Fire 20 rapid requests -- expect 429 (Too Many Requests) after a threshold
for i in $(seq 1 20); do
  curl -s -o /dev/null -w "%{http_code}\n" https://example.com/api/chat \
    -H "Content-Type: application/json" \
    -d '{"message":"test"}'
done
```

If all 20 return 200, rate limiting is absent.

### IDOR on Auto-Generated CRUD

AI code generators produce CRUD endpoints with sequential integer IDs and frequently omit authorization checks. The standard probe: access a resource with your own authenticated session, note the ID, increment it, and confirm the server returns 401 or 403 (not someone else's data).

```bash
# If the API uses integer IDs, test adjacent IDs
curl -s https://example.com/api/users/1
curl -s https://example.com/api/users/2
curl -s https://example.com/api/orders/1001
curl -s https://example.com/api/orders/1002
```

Any 200 response with another user's data is an IDOR vulnerability. This was the second most common critical finding in vibe-coded apps after RLS bypass (DEV community 1,764-app study).

### Remediation Note

Do not tell the site owner to "ask Lovable/Bolt to fix it." Wiz and Autonoma research documents that asking the same AI tool to fix a security issue often introduces a different vulnerability. Provide prescriptive, specific fixes: the exact RLS policy to add, the exact middleware to insert, the exact environment variable to move server-side.

---

## 6. DataForSEO Technology Detection

DataForSEO's technology detection API provides a supplementary signal for stack identification. It returns hosting provider, framework, CDN, analytics tools, and tag managers.

```
call_mcp_tool(
  mcp_name='dataforseo',
  tool_name='domain_analytics_technologies_domain_technologies',
  arguments={"target": "example.com"}
)
```

**Strengths:** Identifies analytics platforms, tag managers, CDN providers, and hosting that bash recon may miss. Useful for inventorying the full technology surface.

**Limitations:** Sparse for static sites. In testing, technicalseonews.com (an Astro site on Vercel) returned only "Vercel" with no framework or CDN detail. The tool depends on its own crawl database and may lag behind recent deployments. It will not detect vibe-coded platform origins, Supabase backends, or framework versions.

**Recommendation:** Run DataForSEO as a supplement to bash recon, never as the primary detection method. Cross-reference its output with the 5-layer bash results. Where they agree, confidence is high. Where DataForSEO returns data that bash did not catch (analytics, tag managers), add it to the stack profile. Where bash caught framework signals that DataForSEO missed, trust bash.

Wappalyzer was archived in 2023. Maintained forks exist (notably Wappalyzergo, a Go library shipping the ~7,200-pattern fingerprint database), but none are available as a reliable runtime API for the audit skill. Do not depend on Wappalyzer-branded services.
