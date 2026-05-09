# Screaming Frog Power Workflows

A practitioner reference for the pre-launch audit skill. Covers the extraction, automation, and integration features that turn Screaming Frog from a crawler into a programmable audit platform. Every section includes the configuration path, the common mistake that wastes the first attempt, and cost or quota realities.

---

## 1. Custom Extraction Patterns

Custom extraction (Configuration > Spider > Extraction) pulls structured fields from every crawled URL into exportable columns. Three modes are available -- XPath, CSS selector, and regex -- each suited to different targets. The critical decision is the extraction type: **Extract Inner HTML** returns the raw markup inside the matched node; **Extract Text** returns visible text only; **Function Value** evaluates an XPath function and returns the result.

### Reference Table

| Target | Expression | Mode | Type | Notes |
|---|---|---|---|---|
| JSON-LD block | `//script[@type='application/ld+json']` | XPath | Extract Inner HTML | Returns raw JSON. Parse downstream. |
| OG title | `//meta[starts-with(@property, 'og:title')]/@content` | XPath | Extract Text | Catches `og:title` without false-matching `og:title:alt`. |
| OG image | `meta[property='og:image']` | CSS | Attribute: `content` | CSS is cleaner here than XPath. |
| Canonical URL | `link[rel='canonical']` | CSS | Attribute: `href` | Returns the href value, not the tag. |
| Hreflang pairs | `//link[@rel='alternate']/@hreflang` and `//link[@rel='alternate']/@href` | XPath | Extract Text | Two separate extractions -- one for language code, one for URL. Cross-reference in the export. |
| Author byline | `//*[contains(@class,'author') or @rel='author'][1]` | XPath | Extract Text | Fragile across CMSes. Add site-specific fallbacks. |
| Publish date | `//meta[@property='article:published_time']/@content` | XPath | Extract Text | Fallback: `//time[@datetime][1]/@datetime` for sites using `<time>` elements instead of meta. |
| H1 count | `count(//h1)` | XPath | **Function Value** | The common mistake: using Extract Inner HTML, which returns the text of the first H1 instead of the count. Function Value evaluates the XPath function and returns the numeric result. Pages with 0 or 2+ H1s are the finding. |

The iPullRank/SFSS-Custom-Extractions GitHub repo maintains a public library of tested extraction patterns. Worth cloning before building from scratch.

For pre-launch audits, wire at minimum: canonical, H1 count, JSON-LD, OG image, and publish date. These five surface the most common staging-to-production regressions (wrong canonical domain, missing schema, placeholder OG images, future-dated articles).

---

## 2. Custom JavaScript Snippets (v20+, May 2024)

The largest capability addition in recent Screaming Frog history. Configuration > Custom > Custom JavaScript. Two snippet types exist, and confusing them is the first mistake practitioners make:

- **Extraction** snippets return data. The return value populates a column in the crawl export. Use for content analysis, scoring, classification.
- **Action** snippets perform steps -- scroll the page, click a cookie banner, wait for lazy-loaded content. They modify the rendered state before extraction runs. No return value.

### Built-in ChatGPT Library

Screaming Frog ships a system library of OpenAI-powered snippets (requires your own API key in Configuration > API Access > OpenAI):

- Meta description generation (from page content)
- Sentiment analysis
- Search intent classification
- Alt text generation (from image URLs)
- Language detection
- JSON-LD generation (from page content)

These are useful starting points but are generic. For audit-specific tasks, the community contributions are more valuable.

### Community Contributions

| Source | What It Does | Notes |
|---|---|---|
| Mike King's "Kermit" | A Custom GPT that generates Screaming Frog JS snippets on demand. Describe the extraction you need in plain English, get a working snippet. | Fastest path from idea to working extraction. |
| iPullRank/SFSS-Custom-Extractions | Public GitHub repo of tested custom extraction and JS snippet patterns. | Clone and adapt rather than writing from scratch. |
| HuestonCo/llmo-optimization-screaming-frog | Batched pointwise scoring for LLM optimization assessment. Uses Gemini API. | Alternative to OpenAI for teams already on Google Cloud. |
| SEM Samurai "clean main content" | Strips nav, footer, cookie banners via GPT-4o-mini to isolate main content for analysis. | Useful precursor to word count or readability scoring. |
| MetaMonster / Ziggy Shtrosberg | JSON-LD generation at scale, meta description generation. Published on the official screamingfrog.co.uk blog. | Closest to production-ready for schema audits. |
| extract-markdown-screaming-frog | Converts page content to Markdown during crawl. | Useful for LLM-readiness assessments and content inventory. |

### Cost Realism

Custom JS snippets that call LLM APIs cost approximately $0.03 per URL on gpt-4o-mini. A 1,000-page site runs about $30. This is real money on a 50,000-page crawl ($1,500), so scope LLM-powered snippets to a filtered URL list (e.g., Internal HTML only, exclude pagination and facets).

**Rate limiting is mandatory.** Set Configuration > Spider > Limits > Max URLs/s to 2 when using OpenAI-powered snippets. Higher rates trigger 429 errors from OpenAI, which Screaming Frog handles poorly -- the extraction silently returns empty, not an error, so you discover the problem only when reviewing exports.

---

## 3. Custom Search Patterns

Configuration > Custom > Search. Regex patterns applied to page HTML or rendered text. Each pattern returns a Contains/Does Not Contain boolean column. The pre-launch audit depends heavily on these for catching content and configuration leaks.

### Reference Table

| Purpose | Pattern | Match Type | Scope |
|---|---|---|---|
| Lorem ipsum / placeholder text | `(?i)lorem ipsum` | Contains | Page Text |
| Staging URLs leaked into content | `(?i)(staging\.|dev\.|test\.|\.local)` | Contains | HTML |
| Phone numbers (US format) | `\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}` | Contains | Page Text |
| Email addresses | `[\w.+-]+@[\w-]+\.[\w.-]+` | Contains | Page Text |
| Missing GA4 tracking | `gtag\|G-[A-Z0-9]+` | Does Not Contain | HTML |
| Hardcoded HTTP resources | `(src\|href)=["']http://` | Contains | HTML |
| Console.log left in production | `console\.(log\|error\|warn)` | Contains | HTML |
| Old brand name (post-rebrand) | `(?i)old brand name` | Contains | Page Text |

**Regex notes:** `(?i)` enables case-insensitive matching. `(?s)` enables dotall / multiline (`.` matches newlines). Screaming Frog uses Java regex syntax, so `\|` is literal pipe within character groups if using alternation within the regex -- use standard `|` for alternation.

The "Does Not Contain" approach for GA4 is the most reliable way to find pages missing analytics. It catches both the gtag.js snippet and the measurement ID.

---

## 4. API Integrations

Configuration > API Access. Each API enriches the crawl with external data, adding columns to the export.

### Google Search Console

Two endpoints, both valuable:

- **URL Inspection API**: 2,000 queries per day, throttled at 600 per minute. Returns index status, crawl date, canonical Google selected, mobile usability, rich results eligibility, and AMP status per URL. This is the single most valuable API integration for indexation audits -- it tells you what Google actually sees, not what you think it should see. The 2,000/day quota means a 10,000-page site takes five days to fully inspect. Plan accordingly.
- **Search Analytics**: Clicks, impressions, CTR, and average position per URL. Pre-launch sites have no data here, but configuring it now means the first post-launch crawl comparison has performance baselines.

### Google Analytics 4

Sessions and pageviews per URL. The primary use case is zombie page detection: URLs with zero sessions over 90 days that are still in the index are candidates for noindex, consolidation, or removal. On a pre-launch site, GA4 has no data. Wire it for the 30-day post-launch follow-up crawl.

### PageSpeed Insights

Lighthouse scores (performance, accessibility, best practices, SEO) plus CrUX field data per URL. Pre-launch sites have no CrUX field data, but the lab scores from Lighthouse are available immediately. Rate-limited -- expect 1-2 seconds per URL. On large sites, sample representative templates rather than crawling every URL through PSI.

### Ahrefs / Majestic / Moz

Backlink metrics per URL: Domain Rating, referring domains, backlinks count, anchor text distribution. Cross-reference traffic with link profile to find pages with strong backlinks but poor content (upgrade candidates) or strong content with no backlinks (promotion candidates). Pre-launch sites have no backlink profile. For migrations, this data is critical -- every URL with backlinks must have a working redirect.

### Pre-Launch API Combo

Wire GSC + PSI + Ahrefs before launch, even though GSC and GA4 return no data. The configuration persists in the `.seospiderconfig` file. When you run the first post-launch crawl with the same config, all APIs fire automatically and the crawl comparison gains performance and indexation dimensions.

---

## 5. Crawl Comparison

Mode > Compare. The killer pre/post-launch feature. Loads two saved crawls and surfaces differences: added URLs, removed URLs, status code changes, redirect changes, canonical changes, indexability changes, schema changes.

### URL Mapping

For cross-domain comparisons (staging.example.com to example.com), configure URL Mapping with a regex find-and-replace. Example: find `https://staging\.example\.com` replace with `https://example.com`. This maps staging URLs to their production equivalents so the comparison is meaningful.

### Change Detection

Configure which elements to track: title, meta description, H1, canonical, word count, status code, indexability. Each changed element surfaces as a row in the comparison export.

### The Pre/Post-Launch Ritual

1. **Crawl staging** with all extractions and API integrations configured. Save the crawl (File > Save).
2. **Launch the site.**
3. **Crawl production** with the identical configuration (load the same `.seospiderconfig`).
4. **Mode > Compare.** Load both crawls.
5. **Red = regression.** Any status code change from 200 to non-200, any indexability change from Indexable to Non-Indexable, any canonical change pointing away from the canonical domain -- these are launch regressions that need immediate attention.

Export the comparison as CSV or XLS for stakeholder review. The comparison report is the single most convincing artifact for proving a clean launch or documenting what broke.

---

## 6. Named Workflows

These are the recurring audit patterns that the pre-launch skill orchestrates. Each is a specific configuration + filter + export sequence.

- **Indexability audit:** Filter to Internal HTML. Sort or filter Indexability column to "Non-Indexable." Categorize by reason (noindex, canonicalized, redirected, client error). Every non-indexable page is either intentional or a bug. Pre-launch, the bug rate is high.
- **Redirect audit:** Reports > Redirect Chains. Any chain longer than two hops costs crawl budget. Chains longer than five may break entirely. Also check Reports > Redirect & Canonical Chains for compounding issues.
- **Internal link audit:** Crawl Analysis > Link Score (Screaming Frog's PageRank-equivalent metric). Export Inlinks and Outlinks tabs. Reports > Anchor Text for distribution analysis. Pages with Link Score below 2 that are commercially important need more internal links.
- **Schema audit:** Enable JSON-LD extraction (Configuration > Spider > Extraction). After crawl, check Reports > Structured Data > Validation Errors and Validation Warnings. Cross-reference with the JSON-LD custom extraction to see what schema types exist per URL.
- **Content audit:** Custom JS snippets for word count, sentiment, and LLM-graded quality. Merge with GA4 sessions and GSC clicks via API integration. Pages with high impressions but low clicks are title/description optimization candidates. Pages with low word count and no traffic are consolidation candidates.
- **Spelling and grammar:** Configuration > Content > Spelling & Grammar (built-in since v17). Surfaces misspellings and grammar issues per URL. Noisy on technical content -- expect false positives on product names and technical terms. Worth running once pre-launch and triaging the top-frequency errors.
- **Orphan pages:** Crawl Analysis after importing GSC URLs, GA4 URLs, and sitemap URLs as additional sources. Orphan URLs are pages that appear in GSC/GA4/sitemap but receive zero internal links from the crawled site. These are invisible to users navigating the site and often invisible to crawlers too.

---

## 7. .seospiderconfig Management

The `.seospiderconfig` file is Screaming Frog's configuration persistence format. Save via File > Configuration > Save As. Load via File > Configuration > Load. A config file captures everything: spider settings, extraction patterns, search patterns, API credentials (encrypted), rendering mode, speed limits, URL exclusions, and export preferences.

### Practitioner Configs

Maintain a library of named configs for recurring audit types:

| Config | Key Settings | When to Use |
|---|---|---|
| `pre-launch.seospiderconfig` | Lorem ipsum + staging URL search patterns, all API integrations wired, JS rendering ON, custom extractions for canonical/H1/schema/OG | Default for this skill |
| `migration.seospiderconfig` | URL mapping regex for old-to-new domain, change detection on all elements, redirect chain reporting | Domain or platform migrations |
| `content.seospiderconfig` | GA4 + GSC APIs, word count extraction, sentiment JS snippet, JS rendering OFF (speed) | Content audits and thin content sweeps |
| `ecommerce.seospiderconfig` | Price and review extraction patterns, faceted URL exclusions (`?sort=`, `?filter=`), query parameter limits | Shopify, WooCommerce, Magento |
| `technical.seospiderconfig` | PSI API, all response code filters, canonical and hreflang extraction, robots.txt analysis | Deep technical audits |
| `link-audit.seospiderconfig` | Ahrefs API, outlinks tab, anchor text report, external link analysis | Backlink and internal link audits |

### CLI Config Loading

Pass a config file on the command line with `--config "path/to/config.seospiderconfig"`. This is the foundation of automated and scheduled crawls.

---

## 8. CLI Automation

Screaming Frog runs headless from the command line, which makes it scriptable, schedulable, and CI/CD-compatible.

### Basic Headless Crawl

```bash
ScreamingFrogSEOSpiderCli \
  --crawl https://example.com \
  --headless \
  --config "/path/to/pre-launch.seospiderconfig" \
  --output-folder "/path/to/output" \
  --timestamped-output \
  --export-tabs "Internal:All" \
  --export-format csv
```

### Crawl from URL List

```bash
ScreamingFrogSEOSpiderCli \
  --crawl-list /path/to/urls.txt \
  --headless \
  --config "/path/to/config.seospiderconfig" \
  --output-folder "/path/to/output" \
  --export-tabs "Internal:All,Response Codes:All" \
  --export-format csv
```

### Key Flags

| Flag | Purpose |
|---|---|
| `--save-crawl` | Saves the crawl database for later comparison or re-analysis |
| `--overwrite` | Overwrites existing output files instead of appending timestamp |
| `--export-format csv\|xls\|gsheet` | Output format. CSV for automation pipelines, XLS for stakeholder reports |
| `--timestamped-output` | Creates a timestamped subdirectory. Prevents overwriting previous runs |
| `--export-tabs "Tab:Filter"` | Specifies which tabs and filters to export. Comma-separated |

### CI/CD Integration

- **Cron (Linux/macOS):** Schedule nightly or weekly crawls with `crontab`. Pipe output to a log file and alert on non-zero exit codes.
- **Cloud Run Jobs (GCP):** Pay per crawl minute. Mount the config file from Cloud Storage. Export results back to Cloud Storage or BigQuery.
- **n8n / Zapier:** Trigger crawls via webhook. n8n can shell out to the CLI directly; Zapier typically triggers a cloud function that runs the CLI.
- **Built-in scheduling:** File > Scheduling in the GUI. Supports daily, weekly, and monthly schedules with auto-export. Simpler than cron but requires the GUI to stay installed and the machine to stay on.

For the pre-launch audit skill, the CLI is used indirectly through the SF MCP server (`mcp__screaming-frog__crawl_site`), which wraps the CLI with structured input/output. The config management and named workflows above apply regardless of whether the crawl is triggered via GUI, CLI, or MCP.
