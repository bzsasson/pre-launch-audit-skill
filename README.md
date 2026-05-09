# Pre-Launch Website Audit

A skill for Claude Code that checks a website before it goes live. It runs 5 sub-audits (technical SEO, AI accessibility, security, performance, and on-page SEO), then tells you what to fix before launch, in priority order.

The idea is simple: a site that launches well is one where all the parts work together. Good performance means nothing if search engines can't crawl the site. Solid SEO means nothing if the site leaks API keys. Clean security headers mean nothing if they break how Googlebot renders the page. The 5 sub-audits aren't independent checklists, they cross-reference each other, deduplicate findings, and surface shared root causes. One fix in the right place often resolves issues across multiple audits.

This was built for SEOs running pre-launch checks, but it's useful for anyone shipping a website, whether that's a developer pushing a side project, a team launching a redesign, or an agency reviewing a client's staging site.

## Where it runs

This is a **Claude Code skill**. It works in:

- **Claude Code CLI** (terminal)
- **Claude Code in VS Code / JetBrains** (IDE extensions)
- **Claude Code on claude.ai** (web at claude.ai/code)

If you haven't used Claude Code before: it's Anthropic's coding agent. Skills are instruction files that teach it how to do specific tasks. This skill teaches it how to audit a website.

## What it covers

The skill runs 5 sub-audits. Each one has its own playbook with specific checks, tool calls, and severity classifications:

- **AI Accessibility**: can AI search engines (ChatGPT Search, Perplexity, Google AI Overviews) see and cite your content? Checks crawler access, CDN/WAF blocking (Cloudflare Bot Fight Mode is a common invisible killer), citation readiness, and whether AI browser agents can navigate your site.
- **Technical SEO**: can search engines find and index your pages? Checks for indexation blockers, JS rendering issues, broken redirects, canonical problems, sitemap validation, staging URL leaks, and more.
- **On-Page SEO**: is the content structured for search visibility? Checks titles, descriptions, heading hierarchy, internal linking, structured data, Open Graph tags, and whether content is formatted for featured snippets and AI citations.
- **Performance**: will the site be fast for real users? Checks Core Web Vitals (LCP, INP, CLS), image optimization, caching, render-blocking resources, and third-party script impact.
- **Security**: are there exposed secrets, missing headers, or known vulnerabilities? Checks for leaked API keys in JS bundles, missing security headers, framework CVEs, and common vibe-coding issues (Supabase RLS, exposed endpoints, IDOR). This is pre-launch hygiene, not a penetration test.

## How it works

Before running any checks, the skill detects your tech stack (which framework, hosting platform, and CDN) using HTTP headers, HTML signatures, DNS records, and bundle paths. It then tailors every check to your specific stack.

Supported stacks include Next.js, Nuxt, Astro, SvelteKit, WordPress, Shopify, Webflow, Framer, Wix, Squarespace, Hugo/Jekyll/Eleventy, Drupal, and AI-generated apps (Lovable, Bolt, Base44, Replit).

For example, a Next.js site on Vercel gets ISR cache validation and `NEXT_PUBLIC_*` env var audits. A WordPress site gets plugin CVE checks and username enumeration detection. A vibe-coded app gets Supabase RLS probing and exposed endpoint sweeps.

## Tools

The skill uses several tools to gather data. It checks which ones are available at startup and works with whatever you have. At minimum, it needs `bash` (curl, dig, openssl, grep), and every sub-audit has a fallback path that uses just command-line tools.

With more tools available, the audit gets deeper:

| Tool | What it adds | Notes |
|---|---|---|
| bash (curl, dig, openssl, grep) | Headers, DNS, TLS, robots.txt, HTML inspection, secret scanning | Always available. The only tool the skill actually needs. |
| [Screaming Frog MCP](https://github.com/bzsasson/screaming-frog-mcp) | Full site crawl, bulk analysis across all URLs, crawl comparison | Optional. Requires a [Screaming Frog](https://www.screamingfrog.co.uk/) paid license (the MCP uses the CLI and database features, which aren't on the free plan). |
| Chrome DevTools (via MCP) | Accessibility tree, Lighthouse scores, performance traces, network waterfall | Optional. The playbooks call Chrome DevTools through an MCP server. You can substitute Playwright (available as a [Claude Code plugin](https://github.com/anthropics/claude-code)) or any browser MCP. |
| [DataForSEO](https://dataforseo.com/) | Lighthouse API, AI search volume data, technology detection | Optional. Paid API (usage-based pricing). Can be replaced with Ahrefs, Semrush, or other SEO data providers. |

**You don't need all of these.** The skill works with just bash. Each additional tool unlocks deeper checks, and Screaming Frog is the biggest upgrade for technical SEO and performance and accessibility checks.

### Swapping tools

The playbooks reference specific tools (Chrome DevTools for browser checks, DataForSEO for Lighthouse data and keyword metrics), but the checks themselves are tool-agnostic, they describe *what* to check, and the tool calls are just one way to get the data. If you use different tools, you can ask Claude to update the playbooks to match your setup. Common swaps: Playwright for Chrome DevTools, Ahrefs or Semrush for DataForSEO.

## Installation

### Plugin marketplace (recommended)

In Claude Code, run:

```
/plugin marketplace add bzsasson/pre-launch-audit-skill
```

This installs the skill automatically using the Claude Code plugin system.

### Manual install

Clone the repo and copy the skill directory:

```bash
git clone https://github.com/bzsasson/pre-launch-audit-skill.git
cp -r pre-launch-audit-skill/skills/pre-launch-audit ~/.claude/skills/pre-launch-audit
```

Or symlink it if you want to pull updates later:

```bash
git clone https://github.com/bzsasson/pre-launch-audit-skill.git
ln -s "$(pwd)/pre-launch-audit-skill/skills/pre-launch-audit" ~/.claude/skills/pre-launch-audit
```

After installation, Claude Code will pick up the skill automatically in new conversations.

## Usage

The skill triggers when you ask Claude Code to audit a site before launch:

- "Run a pre-launch audit on https://example.com"
- "Is this site ready to ship?"
- "Check my staging site before go-live"
- "check if we are ready to launch the new site"

All 5 sub-audits run by default. You can skip any by saying so.

## What you get

A prioritized report with:

- **Top 5 issues** -- the most impactful findings across all sub-audits, with business context
- **P0 (launch blockers)** -- things that cause deindexation, data breaches, or site breakage
- **P1 (fix on launch day)** -- meaningful regressions, significant visibility or security gaps
- **P2 (post-launch)** -- quality improvements, minor gaps
- **P3 (backlog)** -- nice to have, emerging standards
- **What's already good** -- things done right
- **Post-launch monitoring** -- what to set up before launch so you can track performance from day one

Every P0/P1 finding includes what breaks if you don't fix it, the specific fix for your stack (file path, code snippet, or command), and how to verify the fix worked.

## File structure

```
.claude-plugin/
  plugin.json                      # Plugin metadata
  marketplace.json                 # Marketplace catalog
skills/pre-launch-audit/
  SKILL.md                         # Main orchestrator -- controls the audit flow
  audits/
    technical-seo.md               # 10 checks
    ai-accessibility.md            # 7 checks
    security.md                    # 6 areas
    performance.md                 # 8 areas
    on-page.md                     # 9 checks
  references/
    ai-crawler-landscape.md        # Bot taxonomy, user-agents, Cloudflare, llms.txt
    security-checks.md             # Headers, vibe-coding checklist, CVEs, secrets patterns
    sf-power-workflows.md          # Screaming Frog custom extractions, JS snippets, CLI
    performance-budgets.md         # CWV thresholds, diagnostic playbooks, caching
    stack-profiles.md              # 15-stack fingerprints, bash recon, platform ceilings
```

The skill loads playbooks and references on demand -- only the files needed for selected sub-audits are read into context.

## Security scope

The security sub-audit checks for pre-launch hygiene: exposed secrets, missing headers, known CVEs, and common patterns in AI-generated code. It does not perform penetration testing, so no SQL injection fuzzing, no authenticated session abuse, no business-logic exploitation. See `audits/security.md` for the full scope boundary.

## License

MIT
