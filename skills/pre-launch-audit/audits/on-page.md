# On-Page SEO Audit Playbook

Sub-audit module for the pre-launch website audit skill. Covers title tags, meta descriptions, heading hierarchy, content quality and E-E-A-T signals, internal linking, Open Graph and social previews, URL structure, featured snippet and AI citation patterns, and image SEO.

**Primary tools:** SF MCP (if crawl ran), Chrome DevTools (`evaluate_script`), DataForSEO `on_page_content_parsing`
**Fallback:** DataForSEO content parsing + bash curl spot-checks
**Depends on:** SF crawl from Phase 2 (enhanced mode). Can run in degraded mode without SF.

---

## 1. Title Tags

**Why this matters.** The title tag remains the single strongest on-page ranking signal. Google rewrites most titles in 2026 -- roughly 70% of SERPs show a Google-rewritten title. But the 30% it keeps are disproportionately the well-crafted ones: correct length, entity-leading, no keyword stuffing. A rewritten title you did not control is a CTR coin-flip. A title you wrote and Google kept is a CTR asset.

**What to check:**

- **Present on every page.** A missing title is a P0 on any page in the index.
- **Unique across the site.** Duplicate titles cause cannibalization and confuse crawlers.
- **50-60 characters.** Mobile SERP truncation (the canonical viewport since July 2024 mobile-first completion) cuts at approximately 55-60 characters. Aim for 50-60 to keep the full title visible. Titles over 60 characters are not wrong, but you lose control of what the user sees.
- **Primary keyword or entity early.** Front-load the main entity or keyword. Back-loaded keywords get truncated on mobile and carry less weight in title-based ranking signals.
- **Brand suffix consistency.** Pick a format ("| Brand" or "- Brand") and apply it site-wide. Exception: the homepage, where the brand is the primary entity and does not need a suffix.
- **No multiple title tags.** Two `<title>` elements in the head is a parsing ambiguity that guarantees a rewrite.

**SF exports:**

- `Page Titles:Missing` -- pages with no title tag at all
- `Page Titles:Duplicate` -- pages sharing the same title
- `Page Titles:Multiple` -- pages with more than one `<title>` element
- `Page Titles:Over 60 Characters` -- titles exceeding the mobile SERP display threshold

If SF is unavailable, use DataForSEO `on_page_content_parsing` for sampled pages, or bash:
```bash
curl -sL https://example.com | grep -oP '<title[^>]*>\K[^<]+'
```

**What good looks like:**

- Every indexable page has exactly one `<title>`.
- No two pages share a title.
- Titles are 50-60 characters with the primary entity in the first 30 characters.
- Brand suffix is consistent across the site.

**What bad looks like:**

- "Home" or "Untitled" on the homepage.
- Product pages sharing a template title like "Product | MyStore" without the product name.
- Titles stuffed with keywords: "Buy Shoes | Cheap Shoes | Best Shoes | Shoes Online | MyStore".
- Multiple `<title>` tags from conflicting plugins or SSR hydration bugs.

**Consultant interpretation example:**

> Your 4 product category pages all share the title "Shop Our Collection | BrandName." Google will rewrite these, and not in your favor -- it will pull heading text or anchor text from inbound links, which may not match your conversion intent. Each category needs a unique, descriptive title: "Men's Running Shoes | BrandName" rather than a generic template. This is a P1 -- it affects your highest-intent pages.

**Severity:**

| Finding | Severity |
|---|---|
| Missing title on indexable page | P0 |
| Duplicate titles on key pages | P1 |
| Multiple title tags | P1 |
| Titles over 60 chars on commercial pages | P2 |
| Missing brand suffix | P2 |

---

## 2. Meta Descriptions

**Why this matters.** Meta descriptions are not a ranking factor. Google has said this explicitly and repeatedly. They are, however, the primary CTR lever you control on the SERP -- when Google uses them. Google rewrites approximately 70% of meta descriptions in 2026, choosing snippet text from on-page content instead. The 30% it keeps tend to be well-crafted, unique, and accurately summarizing the page. Write for that 30%.

**What to check:**

- **Present on every key page.** Missing descriptions are not a ranking penalty, but they cede CTR control to Google's auto-generated snippet.
- **Unique across the site.** Duplicate descriptions are a signal of template laziness that can trigger snippet rewrites across all affected pages.
- **140-155 characters.** Mobile SERPs truncate around 120-155 characters depending on device and query. Aim for 140-155 to land in the safe zone. Desktop can display slightly more, but mobile is canonical.
- **Include the entity, value proposition, and a soft CTA.** "Learn how to..." or "Compare the top 5..." outperforms a bare keyword list.
- **No keyword stuffing.** Stuffed descriptions are virtually guaranteed to be rewritten.

**SF exports:**

- `Meta Description:Missing` -- pages with no meta description
- `Meta Description:Duplicate` -- pages sharing the same description

If SF is unavailable, use DataForSEO `on_page_content_parsing` or bash:
```bash
curl -sL https://example.com/page | grep -oP '<meta name="description" content="\K[^"]+'
```

**What good looks like:**

- Every landing page, product page, and key content page has a unique meta description.
- Descriptions are 140-155 characters, readable as a sentence, and include the page's primary value proposition.

**What bad looks like:**

- Auto-generated descriptions that are just the first 160 characters of body text.
- Template descriptions: "Welcome to MyStore. We sell products. Visit us today."
- No descriptions at all on conversion-critical pages.

**Severity:**

| Finding | Severity |
|---|---|
| Missing description on high-value pages | P1 |
| Duplicate descriptions | P2 |
| Descriptions over 160 chars | P3 |
| Missing on blog/editorial pages | P3 |

---

## 3. Heading Hierarchy

**Why this matters.** Heading structure serves three audiences simultaneously: users scanning the page, screen readers and browser agents navigating the accessibility tree, and AI citation engines extracting answers. Paragraphs under an H2 whose phrasing matches the user's query are disproportionately cited by AI search engines (ChatGPT Search, Perplexity, Google AI Overviews). Getting headings right is one of the highest-leverage on-page changes for both traditional and AI search visibility.

**What to check:**

- **Exactly one H1 per page.** The H1 should match the user intent for that URL. It is the page's thesis statement.
- **Strict h1 > h2 > h3 nesting.** No skipped levels. An h1 followed by an h3 (skipping h2) breaks the document outline and confuses both screen readers and AI parsers. An h2 followed by an h4 is the same problem.
- **H2s phrased as questions where appropriate.** Question-phrased H2s ("What is X?", "How does Y work?") align with the way AI search engines match user queries to page sections. This is not about keyword stuffing headings -- it is about matching the mental model of the searcher.
- **No heading-level abuse.** Using H2 for visual styling when the content is logically an H4 is a common CMS/theme problem. Headings are semantic, not cosmetic.
- **No empty headings.** `<h2></h2>` or `<h2> </h2>` is invisible to users but creates noise in the document outline.

**SF exports:**

- `H1:Missing` -- pages with no H1 tag
- `H1:Multiple` -- pages with more than one H1

For deeper hierarchy analysis, use Chrome DevTools `evaluate_script`:
```javascript
// Extract heading hierarchy for a page
const headings = Array.from(document.querySelectorAll('h1, h2, h3, h4, h5, h6'));
return headings.map((h, i) => ({
  level: parseInt(h.tagName[1]),
  text: h.textContent.trim().substring(0, 80),
  skipped: i > 0 && parseInt(h.tagName[1]) > parseInt(headings[i-1].tagName[1]) + 1
}));
```

If SF is unavailable, bash:
```bash
curl -sL https://example.com/page | grep -oP '<h[1-6][^>]*>.*?</h[1-6]>' | head -20
```

**What good looks like:**

- One H1 per page, aligned with the page's target query.
- Clean h1 > h2 > h3 nesting with no gaps.
- H2s on informational content phrased as questions or clear noun phrases.
- Each H2 section contains a direct, summarizable first paragraph (the answer-first pattern).

**What bad looks like:**

- Multiple H1s (common in WordPress themes and Shopify templates that put the site name in an H1 in the header).
- H1 > H3 > H2 > H4 scrambled hierarchy.
- H2s used purely for visual styling: large bold text that is semantically a paragraph.
- Generic H2s: "Overview," "Details," "More Information."

**Consultant interpretation example:**

> Your blog template puts the site name in an H1 in the header component, then the article title in a second H1. Every blog post has two H1s. The fix is to change the header site name to a `<span>` or `<p>` with CSS styling -- the H1 belongs to the article title alone. This is a P1 because it affects every blog post (147 pages), confuses the document outline, and dilutes the primary heading signal.

**Severity:**

| Finding | Severity |
|---|---|
| Missing H1 on indexable pages | P1 |
| Multiple H1s (template-level) | P1 |
| Skipped heading levels on key pages | P2 |
| Non-question H2s on informational content | P3 |

---

## 4. Content Quality / E-E-A-T

**Why this matters.** E-E-A-T (Experience, Expertise, Authoritativeness, Trustworthiness) is not a ranking factor in the direct algorithmic sense -- there is no E-E-A-T score in Google's ranking systems. It is a quality rater guideline that shapes the systems Google builds. In practice, the signals that communicate E-E-A-T are measurable and auditable: author identity, publication dates, source citations, topical depth, and content structure. These same signals are also what AI citation engines use to decide which sources to cite.

**What to check:**

### 4a. Word count distribution

- Identify thin content: pages with fewer than 300 words on key landing pages, product pages, or content pages. Thin content is not inherently bad (a contact page does not need 1,000 words), but thin content on pages targeting competitive queries is a signal of low value.
- Use SF custom extraction (word count on body text) or DataForSEO `on_page_content_parsing` for word count.
- Flag pages where the boilerplate (nav, footer, sidebar) outweighs the unique content.

### 4b. Duplicate and near-duplicate content

- SF `Content:Exact Duplicates` and `Content:Near Duplicates` exports.
- Near-duplicates on e-commerce (product variants differing by one attribute) need canonical strategy, not unique content.
- Cross-domain duplicate check: if the site syndicates content, confirm canonical points to the original.

### 4c. Author bylines and Person schema

- **Bylines visible on page.** Author name linked to an author bio page.
- **Person schema** on author pages: `@type: Person` with `name`, `url`, `sameAs` linking to external profiles.
- **Linked external profiles:** LinkedIn, Wikipedia, Wikidata, ORCID (for academic/medical), Crunchbase, X/Twitter. The `sameAs` array in Person schema is how Google's Knowledge Graph connects the author entity to the broader web.
- No author attribution at all is a gap for YMYL (Your Money Your Life) topics and increasingly for any content competing in AI search.

### 4d. Publication and modification dates

- `dateModified` visible in the HTML (not just in schema). Users and AI engines both use visible dates to assess freshness.
- `dateModified` in Article/NewsArticle JSON-LD schema matches the visible date.
- `datePublished` present in schema.
- Stale content (no update in 18+ months on time-sensitive topics) flagged for review.

### 4e. Citations to primary sources

- Outbound links to authoritative primary sources (not just internal links or affiliate links).
- Studies, data sources, official documentation, regulatory filings cited with link.
- Correlation (not causation) with both ranking and AI citation: pages that cite primary sources are more likely to be cited by AI engines themselves.

### 4f. Topical authority -- pillar/cluster model

- Confirm pillar pages exist for each core topic the site targets.
- Confirm spoke/cluster posts link to the pillar page and vice versa.
- Orphan content clusters (posts with no internal links to a pillar) are a missed topical authority signal.
- This check depends on the internal linking analysis in check 5.

**SF exports for content analysis:**

- `Content:Exact Duplicates`
- `Content:Near Duplicates`
- Custom extraction for author bylines (XPath): `//*[contains(@class,'author') or @rel='author'][1]`
- Custom extraction for publish date: `//meta[@property='article:published_time']/@content` and `//time[@datetime][1]/@datetime`

**Consultant interpretation example:**

> Your 12 product guides have no author attribution -- no byline, no author page, no Person schema. For a site selling professional tools, this matters: Google's quality rater guidelines explicitly weight author expertise for product reviews. Add author pages with bios, credentials, and `sameAs` links to LinkedIn profiles. Then add Person schema and visible bylines to every guide. This is a P1 for your review content, P2 for general blog posts.

**Severity:**

| Finding | Severity |
|---|---|
| Thin content (< 300 words) on key landing pages | P1 |
| No author attribution on YMYL or review content | P1 |
| Missing `dateModified` in schema and visible HTML | P2 |
| Duplicate/near-duplicate content without canonical strategy | P1 |
| No outbound links to primary sources on research content | P2 |
| Missing pillar pages for core topics | P2 |

---

## 5. Internal Linking

**Why this matters.** Internal links are the primary mechanism for distributing crawl equity and signaling page importance to search engines. Orphan pages (no internal links pointing to them) are effectively invisible to crawlers that do not rely on the sitemap. Crawl depth (clicks from homepage) directly affects crawl priority. For a pre-launch site, internal linking errors are among the most common and most impactful issues.

**What to check:**

### 5a. Orphan pages

- **Requires SF Crawl Analysis.** This is a post-crawl analysis that compares crawled URLs against sitemap URLs and (if available) GSC/GA4 data to find pages that exist but have no internal links pointing to them.
- Run Crawl Analysis after the main crawl completes.
- Export the "Orphan URLs" report.
- Pre-launch caveat: no GSC or GA4 data exists yet. Orphan detection relies on crawl + sitemap comparison only.
- Every page in the sitemap should be reachable via internal links. A page that exists only in the sitemap is technically findable but signals low importance.

### 5b. Crawl depth distribution

- Priority pages (homepage, main category pages, conversion pages) should be reachable in 3 clicks or fewer from the homepage.
- SF Crawl Depth tab shows distribution.
- Pages at depth 4+ are deprioritized by crawlers and may not be crawled frequently.
- Export `Crawl Depth` column and filter for pages deeper than 3.

### 5c. Link Score (SF PageRank equivalent)

- SF's Link Score is a PageRank-equivalent internal metric.
- Surface commercially important pages with a Link Score below 2. These pages are not receiving enough internal link equity.
- Cross-reference with business priorities: if the highest-value conversion page has a Link Score of 1.2, that is a structural problem.
- Export the Internal tab with Link Score column, sort ascending, and compare against a list of priority pages.

### 5d. Anchor text distribution

- Export SF Inlinks report with Anchor column.
- Pivot in a spreadsheet: group by target URL, count anchor text variants.
- Flag any page where more than 30% of internal anchors use the exact same commercial keyword phrase. This is an over-optimization signal.
- Healthy anchor text is varied and descriptive: "our running shoe guide," "running shoes for beginners," "the complete guide" -- not 47 links all saying "buy running shoes."

**SF exports and tools:**

- Crawl Analysis > Orphan URLs (requires post-crawl analysis run)
- Crawl Depth tab
- Internal tab > Link Score column
- Inlinks report > Anchor column

If SF is unavailable, use bash to spot-check:
```bash
# Check if a specific page has internal links pointing to it
curl -sL https://example.com | grep -c '/target-page/'
curl -sL https://example.com/category/ | grep -c '/target-page/'
```

**Severity:**

| Finding | Severity |
|---|---|
| Orphan pages that are in the sitemap | P1 |
| Priority pages at crawl depth > 3 | P1 |
| Conversion pages with Link Score < 2 | P2 |
| > 30% exact-match anchor on commercial pages | P2 |

---

## 6. Open Graph & Social Previews

**Why this matters.** OG tags control how the page appears when shared on social platforms, messaging apps, and increasingly in AI search result cards. A missing or broken OG image means the share preview is either blank or auto-selected by the platform -- usually a logo or random page image. Staging URLs in OG tags are a classic pre-launch leak that persists in social caches long after launch.

**What to check:**

- **`og:title`** -- present, matches or complements the `<title>`.
- **`og:description`** -- present, matches or complements the meta description.
- **`og:image`** -- present, 1200x630 pixels (the universal safe size), under 8 MB, accessible URL (not behind auth or staging domain).
- **`og:url`** -- present, matches the canonical URL. No staging hostnames.
- **`og:type`** -- present. `website` for the homepage, `article` for content pages.
- **`og:site_name`** -- present, consistent across the site.
- **`twitter:card`** -- set to `summary_large_image` for pages with a hero image.
- **`twitter:site`** -- the site's X/Twitter handle.
- **`twitter:creator`** -- the author's X/Twitter handle (if applicable).
- **No staging URLs.** Check that `og:image`, `og:url`, and any absolute URLs in OG tags use the production domain, not `staging.example.com` or `dev.example.com`.

**Tools:**

SF custom extraction for OG tags:
- `meta[property='og:image']/@content` (CSS selector)
- `meta[property='og:title']/@content`
- `meta[property='og:url']/@content`

Chrome DevTools `evaluate_script`:
```javascript
const ogTags = {};
document.querySelectorAll('meta[property^="og:"]').forEach(m => {
  ogTags[m.getAttribute('property')] = m.getAttribute('content');
});
const twitterTags = {};
document.querySelectorAll('meta[name^="twitter:"]').forEach(m => {
  twitterTags[m.getAttribute('name')] = m.getAttribute('content');
});
return { og: ogTags, twitter: twitterTags };
```

External test: `https://www.opengraph.xyz/` for visual preview validation.

If SF is unavailable, bash:
```bash
curl -sL https://example.com/page | grep -oP '<meta (property|name)="(og:|twitter:)[^"]*" content="[^"]*"'
```

**What good looks like:**

- Every indexable page has `og:title`, `og:description`, `og:image`, `og:url`, `og:type`.
- OG images are 1200x630, load fast, and show meaningful content (not a blank or cropped logo).
- `twitter:card="summary_large_image"` is set on pages with hero images.
- All URLs in OG tags use the production domain.

**What bad looks like:**

- `og:image` pointing to `https://staging.example.com/images/hero.jpg`.
- Missing `og:image` entirely -- social shares show no preview image.
- `og:url` pointing to a different page or the homepage instead of the current page.
- OG image that is 200x200 pixels (renders as a tiny thumbnail instead of a large card).

**Severity:**

| Finding | Severity |
|---|---|
| Staging URLs in OG tags | P0 |
| Missing `og:image` on key pages | P1 |
| Missing `og:title` or `og:description` | P2 |
| Missing `twitter:card` | P2 |
| OG image wrong dimensions | P2 |
| Missing `og:site_name` | P3 |

---

## 7. URL Structure

**Why this matters.** URL structure is a weak direct ranking signal but a strong usability and maintainability signal. Clean URLs are easier to share, easier to read in SERPs, and less likely to break during redesigns. For a pre-launch site, this is the one chance to get URLs right before they accumulate backlinks and become hard to change.

**What to check:**

- **Clean, descriptive slugs.** URLs should describe the page content: `/mens-running-shoes/` not `/cat?id=47&ref=nav`.
- **Lowercase only.** Mixed-case URLs (`/About-Us/`) create duplicate content on case-sensitive servers (Linux, most CDNs). Force lowercase at the edge or in the framework.
- **Hyphen-separated.** Hyphens, not underscores. Google treats hyphens as word separators; underscores join words (`running_shoes` is one token to Google, `running-shoes` is two).
- **5 words or fewer in the slug.** Keep slugs concise. `/blog/2026/01/15/the-complete-ultimate-guide-to-running-shoe-selection-for-beginners/` is unwieldy. `/blog/running-shoe-guide/` is better.
- **No excessive query parameters.** Parameterized URLs (`?sort=price&color=red&page=3`) should be controlled via robots.txt and canonical tags (covered in Technical SEO). Flag uncontrolled parameter sprawl.
- **Consistent patterns across sections.** If blog posts are at `/blog/slug/`, product pages should be at `/products/slug/`, not `/shop/item/slug/` on some and `/store/slug/` on others. Consistency helps users and crawlers predict URL patterns.
- **No stop words unless natural.** `/how-to-tie-a-shoe/` is fine (the stop words are natural). `/the-best-of-the-running-shoes-for-the-road/` has unnecessary stop words.
- **No file extensions in URLs** (unless technically required). `/about.html` is legacy; `/about/` is modern.
- **Trailing slash consistency.** Pick one convention and redirect the other with 301. This is covered in Technical SEO but worth flagging here if inconsistent.

**Tools:**

SF Internal tab -- export all URLs and visually scan for patterns. Filter for URLs containing uppercase, underscores, excessive path depth, or query parameters.

Bash:
```bash
# Check for uppercase URLs
curl -sL https://example.com/sitemap.xml | grep -oP '<loc>\K[^<]+' | grep '[A-Z]'

# Check for query parameters
curl -sL https://example.com/sitemap.xml | grep -oP '<loc>\K[^<]+' | grep '?'

# Check for underscores in paths
curl -sL https://example.com/sitemap.xml | grep -oP '<loc>\K[^<]+' | grep '_'
```

**Severity:**

| Finding | Severity |
|---|---|
| Mixed-case URLs without redirects | P1 |
| Parameterized URLs without canonical/robots strategy | P1 |
| Inconsistent URL patterns across sections | P2 |
| Underscores instead of hyphens | P2 |
| Excessively long slugs (> 5 words) | P3 |
| Stop word bloat | P3 |

---

## 8. Featured Snippet & AI Citation Patterns

**Why this matters.** Featured snippets and AI citations both reward the same content pattern: a clear question, followed by a direct answer, in a structured format. The mechanics differ -- Google extracts snippet content from the page, while AI engines synthesize and cite -- but the content architecture that wins both is identical. Getting this right pre-launch means the content is positioned for both traditional and AI search from day one.

**What to check:**

### 8a. Question-phrased H2s with direct answers

- Informational content should use H2s phrased as questions that match user queries: "What is technical SEO?", "How do I fix a redirect chain?"
- Immediately below each question-phrased H2, the first paragraph should be a direct 40-60 word answer to the question. This is the "answer-first" or BLUF (Bottom Line Up Front) pattern.
- The answer paragraph should be self-contained -- extractable without losing meaning.

### 8b. Numbered lists for process/how-to content

- "How to" intents should use numbered/ordered lists (`<ol>`), not paragraph-form instructions.
- HowTo schema is dead (deprecated September 13, 2023 -- removed from desktop, no rich results). But the content pattern still wins featured snippets. The schema is gone; the content structure remains effective.

### 8c. Tables for comparisons

- Comparison queries ("X vs Y", "best X for Y") should use HTML `<table>` elements, not styled divs.
- Tables are the primary content format Google extracts for comparison featured snippets.
- Tables should have proper `<thead>` and `<th>` markup.

### 8d. TL;DR / "Answer Capsule" at top of articles

- Long-form content benefits from a summary block at the top -- a TL;DR, key takeaways box, or answer capsule.
- This serves both users (who scan) and AI engines (which extract the summary as a citation candidate).
- Can be marked up with a `<aside>` or a visually distinct block. No special schema required.

**Tools:**

Chrome DevTools `evaluate_script`:
```javascript
// Check for question-phrased H2s and their following paragraph
const h2s = Array.from(document.querySelectorAll('h2'));
return h2s.map(h2 => {
  const isQuestion = /^(what|how|why|when|where|who|which|can|do|does|is|are|should|will)\b/i.test(h2.textContent.trim());
  const nextP = h2.nextElementSibling;
  const answerLength = nextP && nextP.tagName === 'P' ? nextP.textContent.trim().split(/\s+/).length : 0;
  return {
    text: h2.textContent.trim().substring(0, 80),
    isQuestion,
    answerWordCount: answerLength,
    hasDirectAnswer: answerLength >= 30 && answerLength <= 80
  };
});
```

**What good looks like:**

- Informational pages have 3-5 question-phrased H2s, each followed by a 40-60 word direct answer.
- How-to content uses numbered lists.
- Comparison content uses real HTML tables with proper header markup.
- Long articles open with a TL;DR or key takeaways section.

**What bad looks like:**

- All H2s are noun phrases on informational content ("Technical SEO" instead of "What is technical SEO?").
- How-to steps written as a single paragraph instead of a numbered list.
- Comparison data in styled `<div>` grids instead of `<table>` elements.
- 3,000-word articles with no summary or TL;DR.

**Severity:**

| Finding | Severity |
|---|---|
| No question-phrased H2s on informational content | P2 |
| How-to content without numbered lists | P2 |
| Comparison content without tables | P3 |
| No TL;DR on long-form content | P3 |

---

## 9. Image SEO

**Why this matters.** Image search drives meaningful traffic for e-commerce, travel, food, and visual industries. Even for B2B or content sites, well-optimized images improve accessibility, page performance (covered in the Performance sub-audit), and the likelihood of appearing in Google Image results and AI-generated visual answers.

**What to check:**

### 9a. Descriptive filenames

- Image filenames should describe the image content: `blue-running-shoe-side.avif`, not `IMG_2391.jpg` or `hero-1.png` or `asset_final_v3.webp`.
- Filenames are a weak signal but cost nothing to get right, and they make the codebase more maintainable.

### 9b. Alt text coverage

- Every meaningful image (photos, charts, diagrams, product images) must have descriptive `alt` text.
- Alt text should describe what the image shows, not stuff keywords: `alt="Side view of the Nike Air Zoom Pegasus 41 in blue"` not `alt="running shoes buy cheap running shoes online"`.
- SF exports: filter images where `Alt Text` column is empty.

### 9c. Decorative image handling

- Decorative images (background textures, spacer images, purely visual elements) should have `alt=""` (empty alt attribute, not missing alt attribute).
- `alt=""` tells screen readers and crawlers to skip the image. A missing `alt` attribute entirely is an accessibility violation.

### 9d. Image sitemap

- If the site is image-heavy (e-commerce, portfolio, media), an image sitemap (`<image:image>` within the XML sitemap) helps Google discover images that might be loaded via JavaScript or CSS backgrounds.
- Not required for text-heavy sites with few images.
- Check that existing sitemaps include image entries if the site relies on image search traffic.

### 9e. Captions

- Where appropriate (editorial content, product detail pages), captions below images provide additional context.
- Google's image SERP considers surrounding text, including captions, when determining image relevance.
- Not every image needs a caption -- but product images, charts, and editorial photos benefit.

**SF exports:**

- Images tab > filter by `Alt Text:Missing` (images with no alt attribute at all)
- Images tab > filter by `Alt Text:Over X Characters` (alt text that is suspiciously long, likely keyword-stuffed)
- Custom extraction for image filenames and paths

Chrome DevTools `evaluate_script`:
```javascript
const images = Array.from(document.querySelectorAll('img'));
return images.map(img => ({
  src: img.src.split('/').pop(),
  alt: img.alt,
  hasAlt: img.hasAttribute('alt'),
  isDecorative: img.alt === '',
  width: img.naturalWidth,
  height: img.naturalHeight
}));
```

If SF is unavailable, bash:
```bash
curl -sL https://example.com/page | grep -oP '<img [^>]*>' | grep -v 'alt=' | head -10
```

**What good looks like:**

- All meaningful images have descriptive alt text.
- Decorative images have `alt=""`.
- Filenames are descriptive and hyphenated.
- Image-heavy sites have `<image:image>` entries in their sitemap.

**What bad looks like:**

- 40% of images missing alt text entirely.
- Alt text that is just the filename: `alt="IMG_2391.jpg"`.
- Keyword-stuffed alt: `alt="best cheap running shoes buy online free shipping sale"`.
- All product images named `product-1.jpg`, `product-2.jpg`, etc.

**Severity:**

| Finding | Severity |
|---|---|
| Missing alt text on product/key content images | P1 |
| Missing alt attribute entirely (not even empty) | P1 |
| Keyword-stuffed alt text | P2 |
| Generic filenames on key images | P3 |
| Missing image sitemap on image-heavy site | P2 |
| No captions on editorial/product images | P3 |

---

## Cross-Connections with Other Sub-Audits

When running this playbook, flag findings that overlap with other sub-audit domains for deduplication in Phase 4:

- **Titles and descriptions** overlap with Technical SEO (duplicate title = canonicalization question).
- **Heading hierarchy** overlaps with AI Accessibility (agent usability depends on semantic headings).
- **OG staging URLs** overlap with Technical SEO staging leak sweep.
- **Image optimization** (format, lazy loading, dimensions) overlaps with Performance sub-audit.
- **Author schema / Person entities** overlap with Technical SEO structured data check.
- **Internal linking** overlaps with Technical SEO crawlability and orphan page detection.

Report each finding once in the final report, noting which sub-audits surfaced it.

---

## Fallback Mode (No SF)

If SF MCP is unavailable, this playbook can still run in degraded mode:

1. **DataForSEO `on_page_content_parsing`** for title, description, headings, word count, and content analysis on sampled pages.
2. **Chrome DevTools `evaluate_script`** for OG tags, heading hierarchy, image alt text, and snippet pattern analysis on individual pages.
3. **Bash `curl`** for spot-checking titles, descriptions, OG tags, and image attributes.
4. **Sitemap parsing** via bash to enumerate URLs for sampling.

Degraded mode cannot perform site-wide analysis (duplicate detection, orphan pages, crawl depth, Link Score, anchor text distribution). Note these gaps explicitly in the report and recommend a full SF crawl post-launch.
