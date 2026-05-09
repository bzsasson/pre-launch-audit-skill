# Security Audit Playbook

**Scope boundary:** Pre-launch hygiene for SEOs, NOT a pentest. The goal is to find the high-yield "this should never have shipped" findings before the site goes live and flag them for the right person to fix.

**Audience:** SEOs running pre-launch audits. This playbook distinguishes between checks an SEO can action directly and checks that need a developer. Security fixes that affect rendering behavior (CSP, CORS, SRI, rate limiting, RLS) are flagged as "hand to dev" because incorrect changes can break what Googlebot sees.

**Primary tools:** bash (curl, grep, dig, openssl), Chrome DevTools
**Fallback:** bash alone covers approximately 80% of checks

---

## How to Read This Playbook

Each finding is tagged:

- **SEO can fix** -- you can action this yourself (DNS records, removing exposed files, flagging in a report)
- **Flag for dev** -- detect the issue, explain the SEO risk, hand to a developer. Do not attempt the fix yourself -- incorrect changes to CSP, CORS, SRI, or auth can break Googlebot's rendered view of the site.
- **Rendering risk** -- this fix can break how Googlebot/AI bots see the page if done incorrectly. The security-harden skill (`/security-harden`) handles these safely.

---

## 1. Transport & DNS

### TLS Certificate -- SEO can fix (verify + flag)

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -issuer -subject -dates
```

**What to verify:**
- Certificate is not expired and has more than 30 days until expiration
- Certificate chain is complete (no missing intermediates)
- TLS 1.0 and 1.1 are disabled; TLS 1.2 is the floor, TLS 1.3 preferred

**Grade target:** A or higher at ssllabs.com/ssltest.

An expired or misconfigured TLS certificate causes browser security warnings that block users and crawlers alike. This is a P0 if broken.

### HSTS -- Flag for dev

```bash
curl -sI https://example.com | grep -i strict-transport
```

**Good value:** `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`

**HSTS preload is irreversible.** Submitting to hstspreload.org bakes the domain into browser source code. Removal takes months and requires a Chrome release cycle. Flag the current state, note whether preload is present, and let the site owner decide. Do not recommend preload submission yourself.

### DNS Authentication -- SEO can fix

```bash
dig +short example.com TXT | grep spf
dig +short _dmarc.example.com TXT
dig +short google._domainkey.example.com TXT
dig +short example.com CAA
```

**What constitutes a pass:**
- **SPF:** `v=spf1` record exists, includes the actual mail provider, ends with `-all` or `~all`
- **DMARC:** `v=DMARC1; p=quarantine` at minimum. `p=none` is monitoring-only and does not protect. Must include `rua=mailto:...`
- **CAA:** At least one `issue` record restricting which CAs can issue certificates

SEOs can add DNS TXT records directly (or instruct the domain registrar). Missing DMARC is a P1 finding -- anyone can spoof emails from the domain, which damages brand trust and deliverability.

### Subdomain Takeover -- SEO can detect, dev fixes

```bash
dig +short subdomain.example.com CNAME
```

If a CNAME points to a deleted Heroku app, S3 bucket, GitHub Pages repo, or Azure CloudApp, flag it. Common vulnerable platforms: Heroku, AWS S3, GitHub Pages, Azure CloudApp, Shopify, Fastly, Netlify.

---

## 2. HTTP Security Headers -- Flag for dev (rendering risk)

Missing security headers are the most common pre-launch finding. **However, incorrectly tightening headers -- especially CSP -- can break Googlebot's rendered view of the page.** Detect and report, but defer fixes to a developer.

### Inspection Command

```bash
curl -sI https://example.com | grep -iE "strict-transport|content-security|x-content-type|x-frame|referrer-policy|permissions-policy|x-powered-by|server:|cross-origin"
```

### What to Report

| Header | What to look for | SEO relevance |
|---|---|---|
| `Content-Security-Policy` | Present? Contains `unsafe-inline`? | **Rendering risk:** removing `unsafe-inline` from script-src can break inline analytics (GTM), structured data injection, and framework rendering. Googlebot enforces CSP. Report the current state, do not recommend changes. |
| `Strict-Transport-Security` | Present with long max-age? | Missing = HTTP fallback possible. Report. |
| `X-Content-Type-Options` | `nosniff` present? | Missing = MIME-type sniffing attacks. Report. |
| `X-Frame-Options` | `DENY` or `SAMEORIGIN`? | Missing = clickjacking risk. Low SEO impact. Report. |
| `Referrer-Policy` | Not `unsafe-url`? | `unsafe-url` leaks full URL to third parties. Report. |
| `Permissions-Policy` | Restricts camera/microphone/geolocation? | Missing = browser features available to embedded content. Report. |
| `X-Powered-By` | Should NOT exist | Reveals framework (Express, Next.js, PHP). **SEO can recommend removal** -- zero rendering risk. |
| `Server` | Should be minimal | `Apache/2.4.54 (Ubuntu)` reveals version info. Report. |

### CORS -- Flag for dev (rendering risk)

```bash
curl -sI -H "Origin: https://evil.example" https://example.com | grep -i access-control
```

`Access-Control-Allow-Origin: *` is common on static sites. **Do not recommend restricting CORS without developer involvement** -- restricting it can break cross-origin font loading (CLS spike, invisible text), client-side API fetches that render content, and third-party embeds. Googlebot's Chromium renderer respects CORS.

Report the current state. If CORS reflects arbitrary origins with `Allow-Credentials: true`, that's a P0 security finding regardless of rendering risk.

### SRI -- Flag for dev (rendering risk)

```bash
curl -sL https://example.com | grep -E '<script[^>]+src="https?://' | grep -v 'integrity='
```

Third-party CDN scripts without SRI are a supply-chain risk (Polyfill.io hijack June 2024, 380k+ sites). **However, adding SRI to mutable CDN URLs can break script loading if the CDN updates the file** -- which breaks Googlebot's rendered view. Report the finding, defer implementation to a developer who can verify CDN immutability.

---

## 3. Secrets Exposure -- SEO can detect

Leaked secrets are the highest-severity finding in a pre-launch audit. SEOs can run these detection commands and report findings. Remediation is straightforward (remove the exposure) but should involve the developer for anything beyond DNS/config.

### Fetch Known Leak Paths -- SEO can run

```bash
for path in .env .env.local .env.production .git/config .git/HEAD package.json .DS_Store docker-compose.yml next.config.js vercel.json netlify.toml; do
  status=$(curl -sI "https://example.com/$path" -o /dev/null -w "%{http_code}")
  if [ "$status" = "200" ]; then
    echo "EXPOSED: /$path (HTTP 200)"
  fi
done
```

**Any 200 response on `.env` or `.git/config` is a P0 launch blocker.** Report immediately.

### Source Maps -- SEO can detect

```bash
curl -sI "https://example.com/_next/static/chunks/main.js.map" -o /dev/null -w "%{http_code}"
curl -sI "https://example.com/assets/index.js.map" -o /dev/null -w "%{http_code}"
```

200 = source maps exposed. P1 finding. Report to dev. Zero SEO risk in the fix (disabling source maps doesn't affect rendering).

### Secret Pattern Scanning in JS Bundles -- SEO can run

```bash
# Download bundled JS files
curl -sL https://example.com | grep -oE 'src="[^"]+\.js"' | grep -oE '/[^"]+' | while read -r js; do
  curl -sL "https://example.com$js"
done > /tmp/all-bundles.js

# Scan for secret patterns
grep -oE 'sk-[A-Za-z0-9]{20,}' /tmp/all-bundles.js             # OpenAI
grep -oE 'sk-ant-[A-Za-z0-9-]{20,}' /tmp/all-bundles.js         # Anthropic
grep -oE 'sk_live_[A-Za-z0-9]{24,}' /tmp/all-bundles.js         # Stripe live key (DEFCON)
grep -oE 'AKIA[0-9A-Z]{16}' /tmp/all-bundles.js                 # AWS access key
grep -oE 'AIza[0-9A-Za-z\-_]{35}' /tmp/all-bundles.js           # Google API key
grep -oE 'xox[baprs]-[A-Za-z0-9-]+' /tmp/all-bundles.js         # Slack token
```

**Severity by key type:**
- `sk_live_` (Stripe live): P0 -- allows charging credit cards on the owner's account
- `sk-` / `sk-ant-` (OpenAI / Anthropic): P0 -- allows unbounded API spend
- `AKIA` (AWS): P0 -- can range from S3 read to full account compromise
- `xox[baprs]-` (Slack): P1 -- workspace access
- `AIza` (Google API): P1 -- unrestricted keys allow billing abuse
- `pk_live_` (Stripe publishable): Acceptable -- designed to be client-side
- `eyJ...eyJ` (JWT): Context required -- Supabase anon JWTs are public by design; service_role JWTs are P0

### Framework Env Var Prefixes -- SEO can check

| Framework | Client-Exposed Prefix | What to look for |
|---|---|---|
| Next.js | `NEXT_PUBLIC_*` | Any variable with SECRET, KEY, PASSWORD, TOKEN in the name |
| Vite / Astro | `VITE_*` / `PUBLIC_*` | Same |
| Nuxt | `NUXT_PUBLIC_*` | Same |

Per Escape.tech (October 2025), 1 in 4 Vercel-hosted vibe-coded apps leaked server secrets via `NEXT_PUBLIC_*`. Report any suspicious-looking env var names to the developer.

### Consultant Interpretation Example

> "I found your Supabase anon key in the client bundle, which is expected -- that is the public key Supabase designed for browser use. However, I also found what appears to be an OpenAI API key (`sk-proj-...`) exposed via `NEXT_PUBLIC_OPENAI_KEY`. This is a P0 blocker: anyone can extract this key and make API calls billed to your account. The developer needs to move the OpenAI call to a server-side API route and remove the `NEXT_PUBLIC_` prefix."

---

## 4. Vibe-Coding Security Checks

AI-generated code has security flaws at 2.74x the rate of human-written code (Stanford/NYU). These checks detect the most common AI-generated vulnerabilities. SEOs can run the detection commands; fixes require a developer.

**Remediation warning:** Do NOT tell the client "ask your AI tool to fix it." Wiz and Autonoma research (2025-2026) documented that asking the same AI tool to fix a security issue often introduces a different vulnerability.

### Checks SEOs Can Run

**Source maps in production** (see Section 3)

**Exposed dotfiles and config** (see Section 3)

**Client-side secrets in bundles** (see Section 3)

**Exposed admin/debug endpoints:**
```bash
for path in admin administrator wp-admin wp-login.php graphql graphiql __graphql debug _debug health metrics actuator api/swagger api-docs openapi.json swagger.json phpinfo.php server-status server-info; do
  code=$(curl -sI "https://example.com/$path" -o /dev/null -w "%{http_code}")
  [ "$code" = "200" ] && echo "ACCESSIBLE [$code]: /$path"
done
```

GraphQL playgrounds (`/graphiql`, `/__graphql`) in production are P1. Debug endpoints (`/debug`, `/actuator`, `/metrics`) expose internal state. Report all accessible paths.

### Checks to Flag for Dev

These require developer expertise to both assess and fix. Detect if the surface area exists and flag for developer review:

**Supabase RLS** -- if `supabase.co` URL appears in client bundle, flag: "Supabase detected in client code. Developer should verify Row Level Security is enabled on all tables containing user data. Reference: CVE-2025-48757 (CVSS 9.3, Lovable/Supabase RLS bypass). **Rendering risk if RLS changes break data loading.**"

**Firebase rules** -- if `firebaseio.com` detected, flag: "Firebase detected. Developer should verify Firestore/RTDB security rules are not world-readable."

**IDOR on API routes** -- if the site has `/api/` routes, flag: "API routes detected. Developer should verify authorization checks on all endpoints, especially those with integer IDs in the URL path. IDOR is the second most common critical vulnerability in AI-generated CRUD apps."

**Rate limiting on AI endpoints** -- if `/api/chat`, `/api/generate`, or similar endpoints exist, flag: "AI proxy endpoints detected. Developer should verify rate limiting is in place. **Rendering risk: overly aggressive rate limiting can throttle Googlebot.**"

**Prompt injection surface** -- if the site has LLM-powered features (chat, AI search, content generation), flag: "LLM integration detected. Developer should verify system prompt boundaries and input sanitization."

**Open redirect** -- if login/redirect flows exist, flag: "Redirect parameters detected. Developer should verify external URLs are rejected."

---

## 5. Framework-Specific CVEs -- SEO can detect, dev fixes

After stack detection, check the framework version against known critical CVEs. SEOs can detect the version; developers apply the update. **Note: major framework version updates can change rendering behavior -- always test post-update.**

### Version Detection

```bash
# Next.js
curl -sI https://example.com | grep -i x-powered-by
curl -sL https://example.com | grep -oE 'next/[0-9]+\.[0-9]+\.[0-9]+'

# WordPress
curl -sL https://example.com | grep -oE 'content="WordPress [0-9.]+"'

# Local project
npm list next react astro nuxt @sveltejs/kit 2>/dev/null
```

### Verified Critical CVEs (as of 2026-05-09)

**CVE-2025-55182: React Server Components Remote Code Execution**
- CVSS 10.0 (Critical). Remote code execution via crafted RSC payload.
- Patched: Next.js 16.0.7, 15.5.7, 15.4.8, 15.3.6, 15.2.6, 15.1.9, 15.0.5
- If unpatched: P0 launch blocker. Flag for dev.

**CVE-2025-54135: CurXecute / Cursor MCP RCE**
- CVSS 8.6. Malicious MCP server can execute code on developer's machine.
- Patched: Cursor 1.3.9
- Not a production site vulnerability, but if the team uses Cursor, flag for dev environment.

**CVE-2026-44578: Next.js SSRF via WebSocket**
- High severity, self-hosted only (not Vercel-hosted).
- If self-hosted Next.js: flag for dev.

### Stack-Specific Checks to Flag

| Stack | What to flag | Who fixes |
|---|---|---|
| Next.js | Server Actions without auth middleware | Dev |
| Astro | `set:html` rendering user input (XSS) | Dev |
| Nuxt | `server/api/` routes without auth (public by default) | Dev |
| SvelteKit | CSRF origin checking disabled | Dev |
| WordPress | `/wp-json/wp/v2/users` returning usernames | Dev |
| WordPress | `xmlrpc.php` accessible (brute force vector) | Dev |

---

## 6. Supply Chain -- SEO can detect, dev fixes

These checks apply when you have access to the local project.

```bash
# Run audit, only flag high and critical
npm audit --audit-level=high

# Check lockfile is committed
git ls-files --error-unmatch package-lock.json 2>/dev/null && echo "Lockfile committed" || echo "WARNING: No lockfile"

# Check for wildcard versions
cat package.json | jq -r '.dependencies // {} | to_entries[] | select(.value == "*" or .value == "latest") | "\(.key): \(.value)"'
```

Any dependency pinned to `"*"` or `"latest"` is a P1 finding. Report to dev.

---

## Severity Reference

| Level | Label | SEO Can Action | Flag for Dev |
|---|---|---|---|
| **P0** | Launch blocker | Exposed `.env`/`.git`, leaked API keys in bundles | Supabase RLS missing, unpatched CVSS 10 CVE, expired TLS |
| **P1** | Fix within 24h | Missing DMARC, source maps exposed, exposed admin endpoints | Missing CSP/HSTS, IDOR on API routes, no rate limiting, framework CVEs |
| **P2** | Post-launch | X-Powered-By present, verbose Server header | Missing Permissions-Policy/COOP/CORP, cookie attributes, npm audit moderate |
| **P3** | Backlog | -- | DNSSEC, advanced CSP reporting, COEP |

---

## Rendering Risk Summary

These security fixes can break Googlebot's rendered view if done incorrectly. Always flag for developer and recommend using the `/security-harden` skill for safe implementation:

| Fix | How it breaks rendering |
|---|---|
| CSP tightening (remove unsafe-inline) | Breaks inline scripts (GTM, analytics, schema injection), inline styles (layout, CLS). Googlebot enforces CSP. |
| CORS restriction | Breaks cross-origin font loading (invisible text, CLS), client-side API fetches, third-party embeds. |
| SRI on CDN scripts | If CDN rotates files, hash breaks, script doesn't load. Content rendered by that script disappears. |
| Rate limiting | Overly aggressive limits throttle Googlebot's crawl requests. |
| Supabase RLS / auth changes | Can break client-side data fetching that renders page content. |
| Framework major version updates | Can change routing, metadata handling, rendering behavior. |
