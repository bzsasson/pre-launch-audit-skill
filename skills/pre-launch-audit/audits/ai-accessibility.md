# AI Accessibility Audit Playbook

**Sub-audit 2 of 5** | Pre-Launch Website Audit Meta-Skill
**Primary tools:** Chrome DevTools (`take_snapshot`, `evaluate_script`, `lighthouse_audit`), bash (`curl`), DataForSEO `ai_optimization_keyword_data_search_volume`
**Fallback:** bash curl + Playwright `browser_snapshot`
**Estimated runtime:** 5-8 minutes

---

## Purpose

This playbook determines whether a site is visible, citable, and usable by AI systems -- training crawlers, AI search engines, user-triggered agents, and autonomous browser agents. The landscape has shifted enough since 2024 that a "good" robots.txt is no longer sufficient. CDN-layer blocking, agentic browsers that ignore robots.txt entirely, and structured data expectations for citation eligibility all demand a broader audit surface.

The auditor's role is to surface the decision, not make it. Whether to allow or block training crawlers is a business decision. The audit provides the information needed to make that decision deliberately rather than by accident.

---

## Check 1: 4-Category Crawler Access Audit

### The Taxonomy

AI bots are not a monolith. Four distinct categories exist, each representing a genuinely different *behavior* with different compliance characteristics and control mechanisms.

| Category | What it does | Examples | Respects robots.txt? | SEO implication of blocking |
|---|---|---|---|---|
| **1. Training** | Crawls to build model weights / training datasets | GPTBot, ClaudeBot, CCBot, Google-Extended, Applebot-Extended, Bytespider, Meta-ExternalAgent, Amazonbot, Cohere-ai, Diffbot, ImagesiftBot, AI2Bot | Yes (most). Google-Extended and Applebot-Extended are granular opt-out controls within this category. | Opt out of training data. Google-Extended blocks Gemini training without affecting Google Search. |
| **2. Search/retrieval** | Powers live AI search answers and citations | OAI-SearchBot, Claude-SearchBot, PerplexityBot, DuckAssistBot, MistralAI-User, Bingbot (powers Copilot) | Mostly | Invisible in AI search results (ChatGPT Search, Claude, Perplexity, Copilot) |
| **3. User-action fetch** | Real-time HTTP fetch on a specific user request | ChatGPT-User, ChatGPT-User/2.0, Claude-User, Perplexity-User, Google-Agent, Google-NotebookLM, Google-Read-Aloud, Meta-ExternalFetcher | **Often ignore robots.txt** when user supplies URL | Needs server-rendered HTML, JSON-LD, semantic markup. Blocking breaks user-initiated requests. |
| **4. AI browsers/agents** | Navigate, click, complete tasks in full Chromium | ChatGPT Operator, ChatGPT Atlas, Perplexity Comet, Google Mariner, Claude Computer Use, Dia, Opera Neon, Gemini-in-Chrome | **Not visible to robots.txt at all** -- standard Chrome UA | Needs accessibility tree, ARIA states, working interactive flows. Detection-only problem. |

Understanding this taxonomy is critical. A site that blocks GPTBot but allows OAI-SearchBot has opted out of training while preserving ChatGPT Search visibility. A site that blocks everything at the CDN layer has no AI presence at all -- which may or may not be the intent.

Note: Google-Extended and Applebot-Extended (previously treated as a separate "opt-out tokens" category in some frameworks) are folded into Training because they represent the same behavior type -- crawling for model weights -- with more granular opt-out control. They don't represent a distinct agent behavior.

### Complete User-Agent Reference

```
# --- OpenAI ---
GPTBot/1.1                              Training crawler
OAI-SearchBot/1.0                       ChatGPT Search index
ChatGPT-User/1.0                        User-triggered fetch (v1)
ChatGPT-User/2.0                        User-triggered fetch (v2)

# --- Anthropic ---
ClaudeBot                               Training + retrieval
Claude-User                             User-triggered fetch
Claude-SearchBot                        Search index (newer)

# --- Google ---
Googlebot                               Traditional search (not an AI bot per se)
Google-Extended                         Gemini/Vertex training opt-out token
Google-Agent                            Agentic fetch
Google-NotebookLM                       NotebookLM fetch
Google-CloudVertexBot                   Vertex grounding
Google-Read-Aloud                       Accessibility/agent reads

# --- Apple ---
Applebot                                Spotlight, Siri, Safari suggestions
Applebot-Extended                       Apple Intelligence opt-out token

# --- Perplexity ---
PerplexityBot                           Index (some non-compliance reported)
Perplexity-User                         User-triggered (ignores robots when URL supplied)

# --- Meta ---
Meta-ExternalAgent                      Meta AI training
Meta-ExternalFetcher                    Link previews + Meta AI fetch

# --- Others ---
CCBot                                   Common Crawl (feeds most open-source LLMs)
Bytespider                              ByteDance (documented non-compliance)
TikTokSpider                            ByteDance
Amazonbot                               Amazon Alexa / shopping
Cohere-ai                               Cohere training
Diffbot                                 Knowledge graph extraction
AI2Bot                                  Allen AI
DuckAssistBot                           DuckDuckGo AI answers
MistralAI-User                          Mistral user-triggered
DataForSeoBot                           DataForSEO
PetalBot                                Aspiegel/Huawei

# --- DEPRECATED (ignore in robots.txt) ---
anthropic-ai                            Deprecated July 2024
claude-web                              Deprecated July 2024
```

Volume context (Cloudflare Radar, January 2026): Googlebot reaches 1.70x more unique URLs than ClaudeBot, 1.76x more than GPTBot, 167x more than PerplexityBot, 714x more than CCBot. AI crawler traffic is approximately 1% of all global web requests, roughly 50 billion requests per day across Cloudflare's network.

### Procedure

**Step 1: Fetch and parse robots.txt**

```bash
curl -s https://example.com/robots.txt
```

For each user-agent in the reference list above, determine:
- Is it explicitly mentioned?
- Is it allowed or disallowed?
- Does it fall under a wildcard rule?
- Are there path-specific restrictions?

Report findings as a table:

| User-Agent | Status | Rule Source |
|---|---|---|
| GPTBot | Disallowed (all) | Explicit `User-agent: GPTBot` block |
| OAI-SearchBot | Allowed (implicit) | No specific rule, falls under `User-agent: *` |
| ClaudeBot | Disallowed (all) | Explicit block |
| ... | ... | ... |

**Step 2: Check meta robots and X-Robots-Tag**

```bash
# Check HTTP headers for X-Robots-Tag
curl -sI https://example.com/ | grep -i x-robots-tag

# Check HTML meta robots
curl -s https://example.com/ | grep -i 'meta.*name.*robots'
```

Look for:
- `noai` -- blocks AI training use
- `noimageai` -- blocks AI image training
- `nosnippetai` -- not honored by Google; note but do not recommend
- `max-image-preview:large` -- required for AI Overviews and Google Discover image eligibility

Note: `noai` and `noimageai` are not universally honored. Google does not support them. They signal intent to compliant crawlers only.

**Step 3: Surface the decision**

Never assume. Present findings and ask:

> Your robots.txt currently [allows/blocks] the following AI crawlers: [list].
> Training crawlers (GPTBot, ClaudeBot, CCBot) are [allowed/blocked].
> Search/retrieval crawlers (OAI-SearchBot, PerplexityBot) are [allowed/blocked].
>
> What is your intended policy?
> - Allow all AI access (maximum visibility)
> - Allow search/retrieval, block training (visibility without data contribution)
> - Block all AI crawlers (no AI presence)
> - Custom (specify per bot)

**Step 4: Flag deprecated entries**

If `anthropic-ai` or `claude-web` appear in robots.txt, note they were deprecated in July 2024. They have no effect. The current Anthropic user-agents are `ClaudeBot`, `Claude-User`, and `Claude-SearchBot`.

### Consultant Interpretation Example

> **Finding:** Your robots.txt blocks GPTBot and ClaudeBot but has no rules for OAI-SearchBot, Claude-SearchBot, or PerplexityBot. These search/retrieval bots inherit the wildcard `Allow: /` rule. Meanwhile, Cloudflare Bot Fight Mode is ON (see Check 2), which blocks all AI bots at the network layer regardless of robots.txt.
>
> **Impact:** You have opted out of training (good if intentional) but are also invisible to AI search engines (likely unintentional). ChatGPT Search, Claude, and Perplexity cannot cite your content.
>
> **Recommendation (P0):** Disable Cloudflare Bot Fight Mode or configure AI Crawl Control to allow search/retrieval bots. Your robots.txt policy is sound but moot when the CDN blocks everything upstream.

---

## Check 2: CDN/WAF Probe

This is the number one silent AI visibility killer. A robots.txt that allows every AI bot is meaningless if the CDN returns 403 or a JavaScript challenge before the bot ever reads it.

### Procedure

**Step 1: Detect CDN from response headers**

```bash
curl -sI https://example.com/ | grep -Ei 'cf-ray|x-vercel-id|x-served-by|x-amz-cf-id|x-nf-request-id|server:|x-wf-page-id|x-wix-request-id|x-shopid'
```

| Header | CDN/Platform |
|---|---|
| `cf-ray` | Cloudflare |
| `x-vercel-id` | Vercel |
| `x-served-by` | Fastly |
| `x-amz-cf-id` | AWS CloudFront |
| `x-nf-request-id` | Netlify |
| `server: Webflow` | Webflow (now on Cloudflare) |
| `x-wix-request-id` | Wix |
| `x-shopid` | Shopify |

**Step 2: AI bot delivery test**

Test with the actual user-agent strings AI bots use. Compare response codes and content length against a normal browser request.

```bash
# Baseline (normal browser)
curl -s -o /dev/null -w "%{http_code} %{size_download}" -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36" https://example.com/

# GPTBot (OpenAI training)
curl -s -o /dev/null -w "%{http_code} %{size_download}" -A "Mozilla/5.0 AppleWebKit/537.36 (KHTML, like Gecko; compatible; GPTBot/1.1; +https://openai.com/gptbot)" https://example.com/

# OAI-SearchBot (ChatGPT Search)
curl -s -o /dev/null -w "%{http_code} %{size_download}" -A "Mozilla/5.0 AppleWebKit/537.36 (KHTML, like Gecko; compatible; OAI-SearchBot/1.0; +https://openai.com/searchbot)" https://example.com/

# ClaudeBot (Anthropic training)
curl -s -o /dev/null -w "%{http_code} %{size_download}" -A "ClaudeBot/1.0" https://example.com/

# PerplexityBot
curl -s -o /dev/null -w "%{http_code} %{size_download}" -A "Mozilla/5.0 (compatible; PerplexityBot/1.0; +https://perplexity.ai/perplexitybot)" https://example.com/

# Googlebot (control -- should always work)
curl -s -o /dev/null -w "%{http_code} %{size_download}" -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" https://example.com/
```

Compare HTTP status codes and response sizes. A 403, a challenge page (small response body with JS), or a significantly smaller response than the browser baseline indicates CDN-level blocking.

**Step 3: Platform-specific checks**

**Cloudflare** (detected by `cf-ray` header):

- **Bot Fight Mode** (free tier): default ON. Blocks automated traffic including AI bots. Cannot be disabled on free tier without upgrading. Check: if GPTBot gets 403 or a Turnstile challenge, this is likely the cause.
- **Super Bot Fight Mode** (Pro+): has an "AI Scrapers and Crawlers" toggle. May be set to block or challenge.
- **AI Crawl Control** (2026): Cloudflare's successor to the blanket block. Allows per-bot configuration, HTTP 402 with `crawler-price` headers (pay-per-crawl, private beta), and Web Bot Auth via HTTP Message Signatures (`Signature-Agent`, `Signature-Input`, `Signature` headers).
- **"Content Independence Day" (July 1, 2025):** All new Cloudflare domains block AI crawlers by default. Existing customers received managed robots.txt suggestions and "block AI bots on monetized pages" controls.

**Webflow** (detected by `server: Webflow` or `data-wf-page` in HTML):

- Mandatory Cloudflare migration deadline: **June 1, 2026**. All Webflow sites move behind Cloudflare.
- Post-migration, Bot Fight Mode is enabled by default. Webflow users cannot directly access Cloudflare dashboard controls without Cloudflare O2O (origin-to-origin) configuration.
- If the site is on Webflow and launching after June 2026, AI bot blocking is the default state unless explicitly configured otherwise.

**WordPress** (detected by `x-pingback`, `/wp-content/`):

- **Wordfence** and **iThemes Security** (now SolidWP) plugins can silently 403 AI bots. Check the response code from the GPTBot curl test. If blocked, the plugin's firewall rules are the likely cause, not the host CDN.
- WordPress 6.3+ includes a native robots.txt AI bot management UI, but it only controls robots.txt directives, not WAF behavior.

**Step 4: Check for HTTP 402 and pay-per-crawl headers**

```bash
curl -sI -A "GPTBot/1.1" https://example.com/ | grep -Ei 'crawler-price|402|signature-agent|signature-input'
```

HTTP 402 (Payment Required) with `crawler-price` headers indicates the site is participating in Cloudflare's pay-per-crawl beta. Note the existence but do not evaluate pricing -- the marketplace is still in private beta.

### Consultant Interpretation Example

> **Finding:** Site is behind Cloudflare (cf-ray present). Curl with GPTBot UA returns 403; curl with Googlebot returns 200 with full content. Browser baseline returns 200. Response body for GPTBot is 1.2 KB (Cloudflare challenge page) versus 47 KB for browser.
>
> **Diagnosis:** Cloudflare Bot Fight Mode or AI Crawl Control is blocking AI crawlers at the network layer. Robots.txt allows GPTBot, but the CDN rejects the request before robots.txt is ever read.
>
> **Impact (P0 if AI visibility is a goal):** This site is completely invisible to ChatGPT Search, Perplexity, Claude, and Copilot. No AI search engine can cite it. Any work on structured data, content optimization, or llms.txt is wasted until this is resolved.
>
> **Fix:** Log into Cloudflare dashboard. Navigate to Security > Bots. Either disable Bot Fight Mode (if on free tier, this requires upgrading to Pro) or configure AI Crawl Control to allow specific bots (OAI-SearchBot, ClaudeBot, PerplexityBot at minimum for search visibility). For Webflow sites post-June 2026, contact Webflow support to request Cloudflare O2O access.

---

## Check 3: AI Search Demand Check

Before investing heavily in AI accessibility, determine whether the site's target keywords actually generate AI search volume. High AI search demand makes every finding in this audit more urgent. Low demand means AI accessibility is a forward investment, not an immediate priority.

### Procedure

**Step 1: Identify target keywords**

From the site's content strategy, SEO targets, or homepage/title tags, extract 5-15 core keywords.

**Step 2: Query DataForSEO via DCL-wrapper**

```
call_mcp_tool(
  mcp_name='dataforseo',
  tool_name='ai_optimization_keyword_data_search_volume',
  arguments={
    "keywords": ["keyword one", "keyword two", "keyword three"],
    "location_code": 2840,
    "language_code": "en"
  }
)
```

Location code 2840 = United States. Adjust for the site's target market.

**Step 3: Interpret results**

- **High AI search volume** (relative to traditional search volume): AI accessibility findings in this audit are P0/P1. Users are actively asking LLMs about these topics. Invisibility means lost traffic and lost citations.
- **Moderate AI search volume**: AI accessibility is P1/P2. The trend matters more than the current number -- AI search volume is growing across most categories.
- **Near-zero AI search volume**: AI accessibility is P2/P3. Still worth implementing fundamentals (they overlap with traditional SEO), but do not prioritize over technical SEO or performance issues.

**Note on LLM mentions tools:** DataForSEO's `ai_opt_llm_ment_*` tools (which show how often a domain is mentioned in LLM responses) require a premium plan. If available, they provide a direct measure of current AI visibility. If access-denied, note the limitation and recommend the client check manually by querying ChatGPT, Claude, and Perplexity with their target keywords.

### Report Format

> **AI Search Demand:** [High / Moderate / Low]
> Top keywords with AI search volume: [list with volumes]
> Implication: [sentence on how this affects priority of remaining checks]

---

## Check 4: Citation Readiness

AI search engines (ChatGPT Search, Perplexity, Google AI Overviews, Copilot, Claude) cite pages that are easy to extract structured answers from. This check evaluates whether the site's content is structured for citation.

The honest position on Generative Engine Optimization (GEO): roughly 80% of what makes a page citable by AI overlaps with what makes it rank well in traditional search. Entity discipline and freshness signals are the incremental 20%. Vendor claims of "340% AI visibility uplift" from proprietary GEO tools are vendor numbers, not independent research. The fundamentals matter most.

### Procedure

**Step 1: Structured data density**

```bash
# Count JSON-LD blocks
curl -s https://example.com/ | grep -c 'application/ld+json'

# Extract and inspect JSON-LD
curl -s https://example.com/ | grep -oP '(?<=<script type="application/ld\+json">).*?(?=</script>)' | python3 -m json.tool 2>/dev/null
```

Or via Chrome DevTools:

```
call_mcp_tool(
  mcp_name='chrome-devtools',
  tool_name='evaluate_script',
  arguments={"expression": "JSON.stringify(Array.from(document.querySelectorAll('script[type=\"application/ld+json\"]')).map(s => JSON.parse(s.textContent)))"}
)
```

What to look for:
- **Organization** schema with `sameAs` links to Wikipedia, Wikidata, LinkedIn, Crunchbase
- **Article** or page-type schema with `author`, `datePublished`, `dateModified`
- **Person** schema for authors with credentials and external profile links
- Schema stacking: pages with 3-4 complementary types (e.g., Article + Organization + BreadcrumbList + FAQPage) are cited roughly 2x more often than pages with a single type
- Pages with 15+ recognized named entities have approximately 4.8x higher citation probability

**Step 2: BLUF (Bottom Line Up Front) pattern**

Check whether content pages follow the answer-first pattern: a direct, summarizable answer within the first 1-2 sentences under each heading, followed by supporting detail. This is the single strongest content-structure signal for AI citation.

Review 3-5 key pages manually or via DevTools snapshot. Look for:
- Question-phrased H2 headings (matches how users query LLMs)
- Direct 40-60 word answer immediately under each H2
- Specific numbers, dates, named entities (high factual density)
- Verifiable claims with cited sources

**Step 3: Freshness signals**

```bash
# Check for datePublished / dateModified in HTML
curl -s https://example.com/example-page | grep -Ei 'datePublished|dateModified|article:published_time|article:modified_time'

# Check sitemap for lastmod
curl -s https://example.com/sitemap.xml | grep -c 'lastmod'
```

Freshness matters disproportionately for AI citation. Perplexity cites from content published within 30 days 82% of the time. Google AI Overviews favor recent content for time-sensitive queries.

- Visible publish and update dates on pages
- `datePublished` and `dateModified` in Article schema
- `lastmod` values in sitemap (must be accurate, not auto-generated to today's date)

**Step 4: IndexNow**

```bash
curl -s https://example.com/IndexNowKey.txt
curl -s https://example.com/.well-known/indexnow
```

IndexNow pings Bing (which powers Copilot) and Yandex about new or updated content. It reduces the delay between publishing and citation eligibility. Check whether the site has it configured. If not, flag as P2 -- easy to implement, meaningful for Bing/Copilot freshness.

**Step 5: Separate real from hype**

The following are **not** ranking or citation factors and should be flagged if the client has invested in them:
- llms.txt as a visibility lever (see Check 6)
- "AI-friendly" content rewriting services that inject FAQ blocks
- Token-stuffing (padding content with keywords for LLM context windows)
- GEO/AEO as a discipline separate from technical SEO fundamentals

The following **are** real, evidence-backed citation signals:
- Entity-rich, factually dense content
- Organization + Article + Person structured data stacking
- External corroboration (Wikipedia, Wikidata, G2, Crunchbase mentions of the same entity)
- Stable URLs (AI citations are URL-based; redirects break attribution)
- Recency and visible freshness signals

---

## Check 5: Agent Usability

Browser agents (ChatGPT Atlas, Perplexity Comet, Claude-in-Chrome, Dia) perceive websites through the accessibility tree -- the same structure screen readers use. A site that is hostile to screen readers is hostile to AI agents. This is not a metaphor. The a11y tree is the primary input these agents consume because it is token-efficient compared to raw HTML.

### Procedure

**Step 1: Capture the accessibility tree**

```
call_mcp_tool(
  mcp_name='chrome-devtools',
  tool_name='take_snapshot',
  arguments={}
)
```

The snapshot returns the accessibility tree as perceived by agents and assistive technology.

Evaluate:

- **Semantic HTML:** Are `<nav>`, `<main>`, `<article>`, `<section>`, `<aside>`, `<footer>` used correctly? Or is everything `<div>` soup?
- **Heading hierarchy:** Does it follow h1 > h2 > h3 strictly without skipping levels? Agents use heading hierarchy as a table of contents for navigation.
- **Labeled form inputs:** Every `<input>` should have a corresponding `<label for="id">`. Placeholder text is not a substitute.
- **Meaningful button text:** Buttons should have descriptive text content or `aria-label`. Icon-only buttons without labels are invisible to agents.
- **Stable selectors:** `data-testid` attributes or semantic IDs provide stable targets for agent interaction. CSS-in-JS hash classes (`.css-1a2b3c`) are fragile and break between builds.

**Step 2: Run Lighthouse audit**

```
call_mcp_tool(
  mcp_name='chrome-devtools',
  tool_name='lighthouse_audit',
  arguments={}
)
```

Lighthouse returns an Accessibility score. This score correlates almost perfectly with agent usability:

| Score | Assessment |
|---|---|
| 95-100 | Agent-friendly. Semantic HTML, proper ARIA, keyboard navigable. |
| 90-94 | Minor issues. Likely some missing labels or contrast issues. |
| 80-89 | Agent-hostile. Significant semantic gaps. Flag as P1. |
| Below 80 | Severely agent-hostile. Flag as P0 if agentic browsing matters. |

**Lighthouse Accessibility score below 90 is a near-perfect proxy for agent-hostility.** This single number captures most of what breaks AI browser agents.

**Step 3: Keyboard navigation test**

Agents navigate via the DOM, which is analogous to keyboard navigation. If a human cannot complete key flows using only the keyboard (Tab, Enter, Escape, arrow keys), an agent cannot either.

Test:
- Can you tab through the navigation and reach all major sections?
- Do interactive elements (buttons, links, dropdowns) respond to Enter/Space?
- Is there a visible focus indicator?
- Can you escape modal dialogs?

**Step 4: Cookie and consent banner evaluation**

Cookie banners that block content until interaction are a major agent failure point. Most browser agents cannot reliably dismiss cookie consent overlays.

Check:
- Does a cookie banner appear?
- Does it block page content (overlay) or sit in a non-blocking position?
- Can it be dismissed via keyboard?
- Does the page render meaningful content without dismissing the banner?

If the banner blocks content and requires JavaScript interaction to dismiss, flag as P1 for agent usability.

**Step 5: Custom div-button audit**

```
call_mcp_tool(
  mcp_name='chrome-devtools',
  tool_name='evaluate_script',
  arguments={"expression": "document.querySelectorAll('div[onclick], span[onclick], div[role=\"button\"]:not([tabindex])').length"}
)
```

Any count above zero indicates interactive elements that lack proper semantics. Agents click DOM elements by role; a `<div onclick="...">` without `role="button"` and `tabindex="0"` is invisible to them.

---

## Check 6: Discoverability Protocols

Several emerging protocols aim to make sites machine-discoverable. Their actual impact ranges from zero to speculative. This check audits their presence and provides an honest assessment of each.

### Procedure

**Step 1: llms.txt**

```bash
curl -s -o /dev/null -w "%{http_code}" https://example.com/llms.txt
curl -s -o /dev/null -w "%{http_code}" https://example.com/llms-full.txt
```

**Honest assessment:** llms.txt has zero proven impact on AI citations. ALLMO's January 2026 study of 94,614 cited URLs found fewer than 1% had llms.txt, with no measurable citation uplift. Google explicitly does not use it (Mueller and Illyes, 2025, reaffirmed January 2026). OpenAI, Anthropic, and Perplexity have made no formal commitment to using it for search ranking.

Where llms.txt is useful: documentation sites and developer-tooling sites where AI coding assistants (Cursor, Claude Code, Copilot) benefit from a structured content index. Mintlify auto-generates it for thousands of docs sites. Anthropic, Cloudflare, Vercel, and Astro publish them.

**Recommendation:** If the site is documentation-heavy or developer-facing, implement it (cheap, defensible, P3). Otherwise, note its absence without concern. Never present as a ranking factor.

**Step 2: ai-agent.json**

```bash
curl -s -o /dev/null -w "%{http_code}" https://example.com/.well-known/ai-agent.json
```

A proposed standard for declaring AI agent capabilities, permissions, and contact information. Extremely early stage. Note presence or absence without urgency. P3.

**Step 3: NLWeb /ask endpoint**

```bash
curl -s -o /dev/null -w "%{http_code}" https://example.com/ask
```

Microsoft's NLWeb protocol exposes a natural-language query endpoint. Relevant primarily for sites that want to function as an AI-queryable data source. Note presence. P3 unless the site explicitly targets this use case.

**Step 4: MCP server**

Check whether the site exposes a Model Context Protocol server for AI tool integration. This is relevant for SaaS products, APIs, and developer tools -- not typical content sites.

```bash
curl -s -o /dev/null -w "%{http_code}" https://example.com/.well-known/mcp.json
```

Note presence or absence. P3 for most sites; P2 for developer tools and SaaS products.

---

## Check 7: Prompt Injection Surface

This is a security overlay specific to AI accessibility. Sites that display user-generated content (UGC) -- comments, reviews, forum posts, profile fields -- create a prompt injection surface when AI agents read and summarize that content.

### Procedure

**Step 1: Identify UGC surfaces**

Does the site have:
- User comments or reviews
- Forum posts
- User profile bios or descriptions
- User-submitted content (recipes, listings, articles)
- Any field where users can input freeform text that is rendered on public pages

If no UGC exists, note this check as not applicable and move on.

**Step 2: Scan for injection vectors**

If UGC is present, check for:

- **Hidden text:** CSS-hidden content (`display: none`, `visibility: hidden`, `font-size: 0`, `color` matching background) that agents will read from the DOM even though humans cannot see it. This is a known vector for injecting instructions into agent context.

```
call_mcp_tool(
  mcp_name='chrome-devtools',
  tool_name='evaluate_script',
  arguments={"expression": "document.querySelectorAll('[style*=\"display:none\"], [style*=\"display: none\"], [style*=\"visibility:hidden\"], [style*=\"font-size:0\"], [style*=\"font-size: 0\"]').length"}
)
```

- **HTML comments:** Agents parsing raw HTML may ingest HTML comments as context. Comments containing instruction-like text (`<!-- ignore previous instructions -->`) have been used in documented attacks.

```bash
curl -s https://example.com/page-with-ugc | grep -c '<!--'
```

- **aria-label overrides:** `aria-label` attributes on elements near UGC can be manipulated if users can inject HTML or if sanitization is incomplete. Agents read `aria-label` preferentially over visible text.

**Step 3: Assess and report**

If UGC exists and any injection vector is present, flag as P1 for security. The documented attack scenarios include:
- Carding and exfiltration via Perplexity Comet (LayerX research, 2025-2026)
- Instruction injection via hidden text that redirects agent behavior
- Brand reputation manipulation via invisible content that agents summarize

Remediation is a security concern, not an SEO concern. Refer to the security sub-audit for specific fixes (input sanitization, CSP, UGC rendering sandboxing).

---

## Defensible Default robots.txt Template

This template represents a balanced stance for a brand site that wants AI search visibility while making a deliberate choice about training. It is a starting point, not a prescription. The three sections reflect the taxonomy: traditional search, AI search/retrieval, and training crawlers.

```
# =============================================================
# Section 1: Traditional Search (Allow)
# =============================================================
User-agent: Googlebot
Allow: /

User-agent: Bingbot
Allow: /

User-agent: Applebot
Allow: /

# =============================================================
# Section 2: AI Search / Retrieval (Allow for citation visibility)
# =============================================================
User-agent: OAI-SearchBot
Allow: /

User-agent: ChatGPT-User
Allow: /

User-agent: Claude-SearchBot
Allow: /

User-agent: Claude-User
Allow: /

User-agent: PerplexityBot
Allow: /

User-agent: Perplexity-User
Allow: /

User-agent: DuckAssistBot
Allow: /

User-agent: MistralAI-User
Allow: /

User-agent: Google-Agent
Allow: /

User-agent: Google-NotebookLM
Allow: /

# =============================================================
# Section 3: Training Crawlers (EXPLICIT DECISION REQUIRED)
#
# Uncomment Allow to contribute to LLM training datasets.
# Leave Disallow to opt out of training while keeping search visibility.
# This is a business decision. Neither choice is wrong.
# =============================================================
User-agent: GPTBot
Disallow: /
# Allow: /

User-agent: ClaudeBot
Disallow: /
# Allow: /

User-agent: Google-Extended
Disallow: /
# Allow: /

User-agent: Applebot-Extended
Disallow: /
# Allow: /

User-agent: Meta-ExternalAgent
Disallow: /
# Allow: /

User-agent: Amazonbot
Disallow: /
# Allow: /

User-agent: Cohere-ai
Disallow: /
# Allow: /

# =============================================================
# Section 4: Block Non-Compliant / Low-Value
#
# Bytespider has documented history of robots.txt non-compliance.
# Server-side blocking (WAF rules by UA) is more reliable.
# CCBot feeds Common Crawl, which trains most open-source LLMs.
# =============================================================
User-agent: Bytespider
Disallow: /

User-agent: TikTokSpider
Disallow: /

User-agent: CCBot
Disallow: /

# =============================================================
# Defaults
# =============================================================
User-agent: *
Disallow: /admin/
Disallow: /api/internal/

Sitemap: https://example.com/sitemap.xml
```

**Important caveats:**
- This robots.txt is only effective for bots that respect it (Categories 1, 2, and 4). Category 3 (user-triggered) often ignores robots.txt. Category 5 (agentic browsers) uses standard Chrome UAs and is invisible to robots.txt entirely.
- For Bytespider and other non-compliant bots, server-side blocking (WAF rules matching the user-agent string) is more reliable than robots.txt.
- This template must be adapted per the client's business decision on training data contribution. The Disallow defaults for training crawlers reflect a conservative starting position.

---

## Severity Guide for This Sub-Audit

| Severity | Condition |
|---|---|
| **P0** | CDN blocking AI bots when AI visibility is a stated goal. Site completely invisible to AI search. |
| **P0** | `noindex` applying to AI search crawlers (blocks citation entirely). |
| **P1** | Robots.txt misconfiguration blocking search/retrieval bots unintentionally. |
| **P1** | Lighthouse Accessibility score below 80 (agent-hostile). |
| **P1** | No structured data on key pages (citation readiness gap). |
| **P1** | Prompt injection vectors on UGC-heavy site. |
| **P2** | Lighthouse Accessibility score 80-89. |
| **P2** | Missing freshness signals (no dateModified, no IndexNow). |
| **P2** | Cookie banner blocks content for agents. |
| **P2** | No Organization/Person schema (entity disambiguation gap). |
| **P3** | llms.txt absent (no proven impact). |
| **P3** | ai-agent.json / NLWeb / MCP server absent. |
| **P3** | Deprecated user-agents in robots.txt (no harm, just noise). |

---

## Fallback Procedures

If tools are unavailable, degrade as follows:

| Missing Tool | Affected Check | Fallback |
|---|---|---|
| Chrome DevTools | Agent Usability (a11y tree, Lighthouse) | Playwright `browser_snapshot` for a11y tree. DataForSEO `on_page_lighthouse` for Lighthouse scores. Manual keyboard navigation test. |
| DataForSEO | AI Search Demand | Skip check. Note that AI search volume data is unavailable. Recommend manual queries to ChatGPT/Claude/Perplexity. |
| Playwright | Secondary fallback | Bash curl covers Checks 1, 2, 4, 6, and 7. Only Check 5 (Agent Usability) requires a browser tool. |
| All MCP tools | Everything | Bash-only mode: curl for robots.txt, headers, CDN detection, llms.txt. Manual review for citation readiness and agent usability. Report limitations clearly. |
