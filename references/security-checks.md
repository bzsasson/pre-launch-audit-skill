# Security Checks Reference

This reference covers the security surface of a pre-launch website audit. The scope is hygiene -- high-yield findings that should never ship to production. Penetration testing (SQLi fuzzing, authenticated session abuse, business-logic exploitation) is explicitly out of scope and called out at the end.

---

## 1. Transport and DNS

A site that ships without solid transport security is broadcasting credentials and session tokens to anyone on the network path. Start here because every other check assumes the channel is trustworthy.

### TLS

Run `testssl.sh https://example.com` locally or check `ssllabs.com/ssltest` remotely. The target is an A grade. Anything below B is a launch blocker. The most common pre-launch failures: TLS 1.0/1.1 still enabled (should be disabled -- they have known padding-oracle and downgrade attacks), expired or misordered intermediate certificates, and missing OCSP stapling. TLS 1.3 should be the preferred protocol; it eliminates an entire round-trip from the handshake and removes legacy cipher suites from negotiation.

### HSTS

The correct header value is:

```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

The `max-age` of 63072000 seconds (two years) is the minimum for HSTS preload list eligibility. `includeSubDomains` prevents downgrade attacks on any subdomain. The `preload` directive signals intent to be added to the browser-shipped HSTS preload list at `hstspreload.org`.

A critical warning: HSTS preload is irreversible in practice. Once a domain is on the preload list, removing it takes months and requires coordinated browser releases. If the site might ever need to serve plain HTTP on any subdomain (internal tools, legacy integrations, development environments), do not submit for preload until every subdomain is confirmed HTTPS-capable. Ship the header without `preload` first, verify nothing breaks, then submit.

### SPF, DKIM, and DMARC

Email authentication prevents domain spoofing in phishing campaigns. Even if the site does not send email, attackers will try to send email as the domain.

```bash
# SPF
dig +short TXT example.com | grep "v=spf1"
# DKIM (replace 'google' with the selector from the provider)
dig +short TXT google._domainkey.example.com
# DMARC
dig +short TXT _dmarc.example.com
```

SPF should enumerate all legitimate senders and end with `-all` (hard fail) or `~all` (soft fail). DMARC policy should be `p=quarantine` at minimum, with a reporting address (`rua=mailto:dmarc-reports@example.com`) to catch spoofing attempts. `p=none` is monitoring-only and does not protect recipients. The goal is `p=reject` eventually, but `p=quarantine` is the launch-day minimum.

### CAA and DNSSEC

CAA records restrict which certificate authorities can issue certificates for the domain:

```bash
dig +short CAA example.com
# Expected: 0 issue "letsencrypt.org" (or your CA)
```

DNSSEC should be enabled where the registrar supports it. It prevents cache-poisoning attacks that could redirect users to a forged copy of the site.

### Subdomain Takeover

Dangling CNAME records pointing to deprovisioned services are one of the most exploitable findings in a pre-launch audit. An attacker claims the orphaned resource and serves arbitrary content on your subdomain.

```bash
subfinder -d example.com -silent | httpx -silent -status-code -title
```

Look for CNAMEs pointing to deprovisioned Heroku apps (NXDOMAIN on `*.herokuapp.com`), deleted S3 buckets (`NoSuchBucket` XML response), unclaimed GitHub Pages (`404 -- There isn't a GitHub Pages site here`), or removed Azure CloudApp instances. Any of these is a P0 finding because an attacker can register the target and serve content under your domain, inheriting cookie scope and trust.

---

## 2. HTTP Security Headers Reference Card

Check headers with `curl -sI https://example.com` or `securityheaders.com`. The table below covers the headers that matter for a pre-launch site.

| Header | Good Value | Bad / Missing | What It Prevents |
|---|---|---|---|
| `Content-Security-Policy` | `default-src 'self'; script-src 'self' 'nonce-xxx'` | Missing entirely, or `unsafe-inline` + `unsafe-eval` together | XSS via injected scripts. Nonce-based CSP is the modern standard. Never combine `unsafe-inline` with `unsafe-eval` -- that disables CSP's primary protection. |
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains; preload` | Missing or `max-age` < 31536000 | SSL stripping / downgrade attacks |
| `X-Content-Type-Options` | `nosniff` | Missing | MIME-type confusion attacks where browsers execute uploaded files as scripts |
| `X-Frame-Options` | `DENY` (or `SAMEORIGIN` if embeds needed) | Missing | Clickjacking. CSP `frame-ancestors 'none'` is the modern equivalent but X-Frame-Options is still needed for older browsers. |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Missing or `unsafe-url` | Leaking full URL paths (including tokens, query params) to third-party origins |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | Missing | Third-party scripts accessing device APIs without consent |
| `Cross-Origin-Opener-Policy` | `same-origin` | Missing | Cross-origin window reference attacks (Spectre-class side channels) |
| `Cross-Origin-Resource-Policy` | `same-site` | Missing | Unauthorized cross-origin embedding of resources |
| `Cross-Origin-Embedder-Policy` | `require-corp` | Missing (only needed if site requires cross-origin isolation) | Spectre-class data leaks; required for `SharedArrayBuffer` |

**Remove these headers** -- they leak stack information to attackers: `X-Powered-By` (exposes framework name and version), verbose `Server` headers (e.g., `Server: Apache/2.4.52 (Ubuntu)`).

**Cookie attributes:** Every cookie must carry `Secure; HttpOnly; SameSite=Lax`. Session cookies should use `SameSite=Strict` where the UX allows it. `SameSite=None` requires `Secure` and should only be used for legitimate cross-site embedding (OAuth flows, embedded widgets). Any cookie missing `HttpOnly` is readable by JavaScript and vulnerable to XSS exfiltration.

---

## 3. CORS, SRI, and Cookie Attributes

### CORS Echo-Origin Vulnerability

The most common CORS misconfiguration is an origin-reflecting server that echoes back whatever `Origin` header it receives, combined with `Access-Control-Allow-Credentials: true`. This lets any attacker site make authenticated requests and read the responses.

```bash
curl -sI -H "Origin: https://evil.example" https://example.com | grep -i access-control
```

If the response contains `Access-Control-Allow-Origin: https://evil.example` with `Access-Control-Allow-Credentials: true`, the site is vulnerable. The browser does prevent `Access-Control-Allow-Origin: *` combined with credentials, but origin-echoing bypasses that protection entirely.

### Subresource Integrity (SRI)

Every third-party script loaded from a CDN must carry an `integrity` attribute:

```html
<script src="https://cdn.example.com/lib.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxLHG..."
        crossorigin="anonymous"></script>
```

The `crossorigin="anonymous"` attribute is required for the browser to perform the integrity check on cross-origin resources.

The precedent that made SRI non-optional: in June 2024, the Polyfill.io domain was acquired by a Chinese entity and began injecting malicious redirects into the `polyfill.min.js` file served to 380,000+ sites. Sites with SRI hashes on their Polyfill.io script tags were protected -- the browser refused to execute the tampered file. Sites without SRI silently served malware to their users. Any CDN-hosted script without SRI is carrying the same class of risk.

### Mixed Content

Any HTTP resource loaded on an HTTPS page is mixed content. Browsers block mixed active content (scripts, iframes) and warn on mixed passive content (images, video). Run Lighthouse or check Chrome DevTools "Issues" panel. Common sources: hardcoded `http://` URLs in CMS content, legacy embed codes, and third-party widgets that have not updated their integration snippets.

---

## 4. Vibe-Coding 11-Point Checklist

AI-assisted code generation has created a distinct security archetype. The numbers are stark: AI-assisted code has security flaws at 2.74x the rate of human-written code (Stanford/NYU study). 40% of Copilot-generated code contained security vulnerabilities in controlled studies. The secret-leak rate for AI-assisted commits is 3.2% compared to a 1.5% baseline. In 2025 alone, 28.65 million new hardcoded secrets were committed to public GitHub repositories, a 34% year-over-year increase.

Run every check below on any site built with Cursor, Copilot, Lovable, Bolt, Base44, Replit, or similar AI-assisted tools.

**1. Source maps in production.** Production source maps expose the entire codebase including comments, TODOs, and internal logic.

```bash
# Next.js
curl -sI "https://example.com/_next/static/chunks/main-*.js.map"
# Vite
curl -sI "https://example.com/assets/index-*.js.map"
```

Any 200 response is a P0 finding.

**2. Exposed dotfiles and VCS.** AI tools routinely generate configuration files and forget to exclude them from deployment.

```bash
for path in /.env /.env.local /.env.production /.git/config /.git/HEAD /.DS_Store /package.json /docker-compose.yml /vercel.json /netlify.toml; do
  code=$(curl -sI -o /dev/null -w '%{http_code}' "https://example.com$path")
  [ "$code" = "200" ] && echo "EXPOSED: $path"
done
```

**3. Client-side secrets in JS bundles.** Download the production JavaScript and grep for secret patterns (see section 7 for the full regex library). `trufflehog filesystem ./dist` or `gitleaks detect --source ./dist` automate this.

**4. Supabase RLS audit.** See section 5 for the full workflow.

**5. Firebase rules.** If `firebaseio.com` or Firestore SDK is present, test unauthenticated read access: `curl https://PROJECT.firebaseio.com/.json`. Any data returned means the database is world-readable.

**6. NEXT_PUBLIC_* audit.** Every variable prefixed with `NEXT_PUBLIC_` is compiled into the client bundle. Escape.tech's October 2025 study found 1 in 4 Vercel-hosted vibe-coded apps leaked server secrets this way. Grep the built JS for any `NEXT_PUBLIC_` value that contains API keys, database URLs, or internal service credentials.

**7. Exposed admin and debug endpoints.** Check: `/admin`, `/graphql`, `/graphiql`, `/__graphql`, `/debug`, `/health`, `/metrics`, `/actuator`, `/api/swagger`, `/api-docs`, `/openapi.json`, `/swagger.json`, `/phpinfo.php`, `/server-status`.

**8. IDOR sweep.** Any URL of shape `/api/{resource}/{id}` where `{id}` is an integer: increment the ID and confirm 401/403, not 200 with another user's data. AI-generated CRUD endpoints frequently omit authorization middleware.

**9. Open redirect.** Test parameters like `?next=`, `?redirect=`, `?url=`, `?return_to=` with an external host. The endpoint should reject external URLs.

**10. Rate limiting on AI endpoints.** Any `/api/chat`, `/api/generate`, or `/api/completions` endpoint should rate-limit unauthenticated requests. Fire 20 rapid requests and expect 429 by the end. Without rate limiting, anyone can drain the site's OpenAI or Anthropic budget.

**11. Prompt injection surface.** For any site with LLM features, document where untrusted user input enters the prompt. Confirm a system-prompt boundary exists (delimiters, structured tool calls, or guardrails like Lakera or PromptArmor).

**Remediation warning:** Wiz and Autonoma research (2025-2026) found that using the same AI tool to fix vulnerabilities it introduced frequently creates new vulnerabilities in the fix. When remediating vibe-coded security findings, the fixes should be reviewed by a human or a different analysis tool than the one that generated the vulnerable code.

---

## 5. Supabase RLS Probing Workflow

Supabase exposes a PostgREST API at `$SUPABASE_URL/rest/v1/`. The anon key is public by design (it is in the client bundle), but Row Level Security policies must restrict what that key can access. When RLS is missing or misconfigured, the anon key becomes a full-access database credential.

**Step 1: Extract the Supabase URL and anon key from the client bundle.**

```bash
curl -sL https://example.com | grep -oE 'https://[a-z0-9]+\.supabase\.co'
curl -sL https://example.com -o bundle.html
grep -oE 'eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+' bundle.html
```

**Step 2: Decode the JWT and check the role claim.**

```bash
echo "$ANON_KEY" | cut -d. -f2 | base64 -d 2>/dev/null | jq .role
```

If the role is `service_role` instead of `anon`, stop immediately -- this is a catastrophic finding. The service_role key bypasses all RLS policies and has full database access. The Moltbook breach (1.5 million API keys and 35,000 email addresses exposed) was caused by embedding a service_role key in the client bundle.

**Step 3: List available tables.**

```bash
curl -s "$SUPABASE_URL/rest/v1/" \
  -H "apikey: $ANON_KEY" \
  -H "Authorization: Bearer $ANON_KEY" | jq .
```

**Step 4: Probe each table for unprotected data.**

```bash
for table in users profiles orders messages payments documents; do
  echo "--- $table ---"
  curl -s "$SUPABASE_URL/rest/v1/$table?limit=1" \
    -H "apikey: $ANON_KEY" \
    -H "Authorization: Bearer $ANON_KEY"
  echo
done
```

Any 200 response with row data on a table that should be private means RLS is missing or broken. This is the CVE-2025-48757 class of vulnerability (CVSS 9.3), which was found across 303 vulnerable endpoints spanning 170+ Lovable-generated applications. The root cause is consistent: AI code generators create tables and CRUD endpoints but do not enable or configure RLS policies.

**Step 5: Test write access.**

```bash
curl -s "$SUPABASE_URL/rest/v1/profiles" \
  -H "apikey: $ANON_KEY" \
  -H "Authorization: Bearer $ANON_KEY" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=minimal" \
  -d '{"id": "test-audit-probe", "name": "audit"}' \
  -w "\n%{http_code}"
```

A 201 response means unauthenticated users can insert rows. Clean up any test data created during the probe.

---

## 6. Framework CVE Lookup

The methodology is straightforward: identify the framework and version from the stack detection phase, then check the NVD and the framework's security advisories.

```bash
# If you have access to the project source
npm ls next
npm ls react
npm audit --production
```

### Verified CVEs (as of 2026-05-09)

**CVE-2025-55182 -- React Server Components Remote Code Execution.** CVSS 10.0. Affects React Server Components in Next.js. An attacker can achieve RCE through crafted RSC payloads. Patched in Next.js 16.0.7, 15.5.7, 15.4.8, 15.3.6, 15.2.6, 15.1.9, and 15.0.5. Also patched in React 19.0.1, 19.1.2, and 19.2.1. Any Next.js site using App Router with RSC on an unpatched version is critically vulnerable.

**CVE-2025-54135 -- CurXecute (Cursor MCP RCE).** CVSS 8.6. A malicious MCP server could achieve remote code execution through Cursor's MCP integration. Patched in Cursor 1.3.9. Relevant to the audit if the site was built with Cursor and the development environment is in scope.

**CVE-2026-44578 -- Next.js SSRF via WebSocket.** Affects self-hosted Next.js deployments only (not Vercel). An attacker can exploit WebSocket upgrade handling to perform server-side request forgery. Check if the deployment is self-hosted before flagging.

**CVE-2026-44573 -- Next.js Pages Router i18n data-route bypass.** Affects Next.js Pages Router with internationalization configured. Allows bypassing access controls on data routes through crafted i18n path segments.

### Stack-Specific Version Checks

| Stack | Check Command | Security Advisory Source |
|---|---|---|
| Next.js | `npm ls next` | `github.com/vercel/next.js/security/advisories` |
| React | `npm ls react react-dom` | `github.com/facebook/react/security/advisories` |
| WordPress | Admin dashboard or `wp-includes/version.php` | `wpscan.com/wordpresses` |
| Nuxt | `npm ls nuxt` | `github.com/nuxt/nuxt/security/advisories` |
| Astro | `npm ls astro` | `github.com/withastro/astro/security/advisories` |
| Django | `pip show django` | `djangoproject.com/weblog/` (security category) |
| Rails | `bundle show rails` | `rubyonrails.org/security/` |
| Laravel | `composer show laravel/framework` | `github.com/laravel/framework/security/advisories` |

---

## 7. Secrets Regex Pattern Library

These patterns catch the most common leaked credentials in client-side JavaScript bundles. Run them against the production build output, not the source repository (secrets may be injected at build time and absent from source).

| Pattern | Matches | Severity |
|---|---|---|
| `sk-[A-Za-z0-9]{20,}` | OpenAI API key | Critical -- full API access, billing exposure |
| `sk-ant-[A-Za-z0-9-]{20,}` | Anthropic API key | Critical -- full API access, billing exposure |
| `sk_live_[A-Za-z0-9]{24,}` | Stripe live secret key | DEFCON -- can charge cards, issue refunds, access all payment data |
| `pk_live_` | Stripe publishable key | Acceptable -- designed to be client-side, can only create tokens |
| `AKIA[0-9A-Z]{16}` | AWS access key ID | Critical -- scope depends on attached IAM policy, often over-permissioned |
| `AIza[0-9A-Za-z\-_]{35}` | Google API key | High -- scope depends on API restrictions, often unrestricted |
| `xox[baprs]-` | Slack bot/app/user token | Critical -- workspace access, message history, file downloads |
| `eyJ[A-Za-z0-9_-]+\.eyJ` | JWT token | Context required -- Supabase anon JWTs match this pattern and are intentionally public; service_role JWTs matching this pattern are critical |

### Framework Environment Variable Prefixes

These prefixes mark variables that are compiled into client-side bundles by design. Any server secret using one of these prefixes will be exposed to every visitor.

| Prefix | Framework | Risk |
|---|---|---|
| `NEXT_PUBLIC_*` | Next.js | Compiled into client bundle at build time |
| `VITE_*` | Vite (SvelteKit, Nuxt 3, custom) | Compiled into client bundle at build time |
| `PUBLIC_*` | SvelteKit | Available in client code |
| `NUXT_PUBLIC_*` | Nuxt 3 | Exposed to client via runtime config |
| `GATSBY_*` | Gatsby | Embedded in static HTML/JS at build time |
| `REACT_APP_*` | Create React App | Embedded in client bundle at build time |
| `EXPO_PUBLIC_*` | Expo / React Native | Embedded in client bundle |

The audit should grep the production bundle for any of these prefixes followed by patterns from the secrets table above. The combination -- a framework "public" prefix wrapping a secret key -- is the canonical vibe-coding leak.

---

## 8. Tooling for Non-Pentest Pre-Launch

These tools are safe to run against a production or staging site without causing disruption or triggering incident response. They perform passive analysis, not active exploitation.

**nuclei** is the highest-value tool for this scope. Run with the `http/exposures` and `http/misconfiguration` template directories for fast, low-false-positive detection of exposed files, open redirects, misconfigurations, and known CVE signatures. It is opinionated about what constitutes a finding and rarely wastes time on noise.

```bash
nuclei -u https://example.com -t http/exposures/ -t http/misconfiguration/ -silent
```

**nikto** is older and noisier. It checks for thousands of known dangerous files and misconfigurations but produces more false positives on modern stacks. Useful as a second pass if nuclei is clean and you want confirmation.

**OWASP ZAP baseline** (`zap-baseline.py`) runs a passive-only scan. It spiders the site, observes responses, and flags header issues, cookie problems, and information leakage without sending any attack payloads.

**testssl.sh** is the definitive TLS analysis tool. It checks cipher suites, protocol versions, certificate chains, known vulnerabilities (BEAST, POODLE, Heartbleed, ROBOT), and HSTS configuration in a single run.

**trufflehog** and **gitleaks** scan for secrets. Run them against the deployed build artifacts (`dist/`, `.next/`, `build/`), not just the git history. Secrets injected at build time through environment variables will not appear in the repository but will be in the output bundles.

**dnstwist** enumerates typosquat and homoglyph domains that could be used for phishing. Run before launch to identify domains an attacker might register to impersonate the brand.

```bash
dnstwist --registered example.com
```

**Orchestrated subdomain hygiene chain:**

```bash
subfinder -d example.com -silent | httpx -silent -status-code | nuclei -t http/exposures/ -silent
```

This pipeline enumerates subdomains, checks which are live, and scans them for exposed files and misconfigurations. It catches forgotten staging environments, deprovisioned services with dangling DNS, and shadow IT subdomains that nobody remembers deploying.

---

## 9. Scope Boundary

**In scope for this audit:** transport security, DNS hardening, HTTP security headers, CORS configuration, SRI on third-party scripts, cookie attributes, secret exposure in client bundles, framework version CVEs, vibe-coding hygiene (RLS, IDOR, exposed endpoints, source maps, dotfiles), supply-chain integrity, and subdomain takeover. These are configuration and deployment hygiene findings -- things that should never ship to production regardless of the application's business logic.

**Out of scope:** SQL injection and XSS fuzzing with sqlmap or similar active exploitation tools, authenticated session abuse (privilege escalation, session fixation), business-logic exploitation (coupon stacking, race conditions in checkout), and any testing that requires creating accounts or manipulating application state. Those are penetration testing activities that require a formal scope agreement, rules of engagement, and often a separate security team.

The line is drawn at passive observation and safe probing versus active exploitation. Everything in this reference can be run by a site owner or their consultant against their own property without risk of disruption, data loss, or legal ambiguity. Everything beyond it requires explicit authorization and a different engagement model.
