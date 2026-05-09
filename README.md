# Pre-Launch Website Audit

A Claude Code skill that runs a comprehensive pre-launch website audit across 5 domains: technical SEO, AI accessibility, security, performance, and on-page SEO.

It detects the site's tech stack, tailors checks to the framework, runs sub-audits in parallel where possible, and delivers a prioritized report with stack-specific fix recommendations.

## What it covers

- **Technical SEO** -- indexation blockers, JS rendering, crawlability, canonicalization, sitemaps, redirects, structured data, hreflang, staging leak sweeps, pagination
- **AI Accessibility** -- 4-category crawler access audit, CDN/WAF probing (Cloudflare Bot Fight Mode, etc.), citation readiness, agent usability via a11y tree, discoverability protocols (llms.txt, ai-agent.json)
- **Security** -- transport/DNS, HTTP security headers, secrets exposure, vibe-coding checks (Supabase RLS, IDOR, exposed endpoints), framework CVEs, supply chain
- **Performance** -- Core Web Vitals (LCP/INP/CLS), Lighthouse, network waterfall, caching/CDN, image optimization, render-blocking resources, modern primitives (Early Hints, Speculation Rules, bfcache)
- **On-Page SEO** -- titles, descriptions, heading hierarchy, content quality/E-E-A-T, internal linking, Open Graph/social, URL structure, featured snippet and AI citation patterns, image SEO

## Stack detection

The skill fingerprints the site's framework, hosting, and CDN before running any checks. Supported stacks include Next.js, Nuxt, Astro, SvelteKit, WordPress, Shopify, Webflow, Framer, Wix, Squarespace, Hugo/Jekyll/Eleventy, Drupal, and vibe-coded platforms (Lovable, Bolt, Base44, Replit). Each stack gets tailored checks injected into the relevant sub-audits.

## Tools it uses

The skill probes for available tools at startup and degrades gracefully when tools are missing. Every sub-audit has a minimum viable path using just bash.

| Tool | What it provides |
|---|---|
| Screaming Frog MCP | Full site crawl, custom extractions, crawl comparison |
| Chrome DevTools | Accessibility tree, Lighthouse, performance traces, network waterfall |
| DataForSEO | Lighthouse API, AI search volume, technology detection |
| Playwright | Browser snapshots (fallback for DevTools) |
| Ahrefs MCP | Backlink metrics, keyword data |
| bash | curl, dig, openssl, grep -- always available |

## Installation

Copy the skill into your Claude Code skills directory:

```bash
# Clone the repo
git clone https://github.com/bzsasson/pre-launch-audit-skill.git

# Copy to your skills directory
cp -r pre-launch-audit-skill ~/.claude/skills/pre-launch-audit
```

Or if you prefer to keep it as a git repo for updates:

```bash
ln -s /path/to/pre-launch-audit-skill ~/.claude/skills/pre-launch-audit
```

## Usage

In Claude Code, the skill triggers when you ask to audit a site before launch:

- "Run a pre-launch audit on https://example.com"
- "Is this site ready to ship?"
- "Check my staging site before go-live"
- "Audit https://example.com for technical SEO, security, and performance"

All 5 sub-audits run by default. You can skip any by telling it which ones to drop.

## Output

The skill delivers a prioritized report:

- **Top 5 issues** with business impact and specific fixes
- **P0 (launch blockers)** -- deindexation, data breach, site breakage
- **P1 (launch day)** -- meaningful regressions, significant gaps
- **P2 (post-launch)** -- quality improvements
- **P3 (backlog)** -- nice to have, emerging standards
- **What's already good** -- things done right
- **Post-launch monitoring setup** -- what to wire before launch

Every P0/P1 finding includes the business impact, a stack-specific fix (exact file path, code snippet, or command), and a verification command.

## File structure

```
SKILL.md                           # Main skill orchestrator
audits/
  technical-seo.md                 # 10 checks
  ai-accessibility.md              # 7 checks
  security.md                      # 6 areas
  performance.md                   # 8 areas
  on-page.md                       # 9 checks
references/
  ai-crawler-landscape.md          # Bot taxonomy, user-agents, Cloudflare, llms.txt, GEO
  security-checks.md               # Headers, vibe-coding checklist, CVEs, secrets patterns
  sf-power-workflows.md            # Custom extractions, JS snippets, CLI automation
  performance-budgets.md           # CWV thresholds, diagnostic playbooks, caching
  stack-profiles.md                # 15-stack fingerprints, bash recon, platform ceilings
```

The skill loads playbooks and references on demand -- only the files needed for selected sub-audits are read.

## Security scope

The security sub-audit covers pre-launch hygiene: headers, secrets exposure, known CVEs, and vibe-coding patterns. It is not a penetration test. No SQLi/XSS fuzzing, no authenticated session abuse, no business-logic exploitation. See `audits/security.md` for the full scope boundary.

## License

MIT
