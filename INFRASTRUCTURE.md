# Infrastructure

Operational reference for the DayFunk and Camp DayFunk web properties. Covers DNS, redirects, hosting, and Search Console. Update this file when infrastructure changes.

## Domains

| Domain | Role | Registrar | DNS |
|--------|------|-----------|-----|
| `dayfunk.com` | Canonical site | (verify in Cloudflare) | Cloudflare |
| `www.dayfunk.com` | Redirects to apex | — | Cloudflare |
| `campdayfunk.com` | Legacy / redirects to dayfunk.com | (verify in Cloudflare) | Cloudflare |
| `www.campdayfunk.com` | Redirects to dayfunk.com | — | Cloudflare |

Both zones are managed in Cloudflare under the same account.

## Hosting

The site is a single-page static HTML/CSS build with no build step. Source lives in GitHub at `craigspree/dayfunk-website` and deploys automatically to Cloudflare Pages on every commit to `main`.

- Pages project name: `dayfunk-website`
- Production URL: `dayfunk-website.pages.dev`
- Deploy trigger: push to `main` branch on GitHub
- No CI checks, no build command, no environment variables required

Custom domains attached to the Pages project: `dayfunk.com` and `www.dayfunk.com` only. The campdayfunk domains are NOT attached as Pages custom domains. They are handled exclusively by redirect rules in the campdayfunk.com Cloudflare zone (see Redirects below). Adding them to Pages custom domains will break the redirects, because Pages will intercept the request before the redirect rule fires.

## DNS records

### dayfunk.com zone

- `dayfunk.com` (apex) → CNAME to `dayfunk-website.pages.dev`, **proxied (orange cloud)**
- `www.dayfunk.com` → CNAME to `dayfunk-website.pages.dev`, **proxied (orange cloud)**
- TXT records for Google Search Console verification → DNS only (grey cloud)
- DKIM, SPF, DMARC records as needed for email (verify current state in Cloudflare DNS)

### campdayfunk.com zone

- `campdayfunk.com` (apex) → CNAME to `dayfunk-website.pages.dev`, **proxied (orange cloud)**
- `www.campdayfunk.com` → CNAME to `dayfunk-website.pages.dev`, **proxied (orange cloud)**
- TXT record for Google Search Console verification → DNS only (grey cloud)

The proxied (orange cloud) state is required on the campdayfunk records. Without it, traffic bypasses the Cloudflare edge and the redirect rules never fire.

## Redirects

All redirects are configured as Cloudflare Redirect Rules at the zone level. There is also a `_redirects` file in the repo as a fallback, but the Cloudflare edge rules are the primary mechanism.

Important: Cloudflare Redirect Rules only apply to traffic within the zone they are created in. A rule in the dayfunk.com zone cannot intercept requests arriving at campdayfunk.com. Always create the rule in the correct zone.

### dayfunk.com zone rules

1. `www.dayfunk.com/*` → `https://dayfunk.com/$1` (301, preserve query string)

### campdayfunk.com zone rules

1. `/game` path → (existing rule, placed first, do not modify without context)
2. Hostname equals `campdayfunk.com` OR `www.campdayfunk.com` → target `concat("https://dayfunk.com", http.request.uri.path)`, 301, preserve query string, **place at: Last**

Order matters. The catch-all campdayfunk redirect must be ordered LAST so the `/game` rule fires first.

## Google Search Console

Two properties are verified and active:

- `dayfunk.com` (URL prefix property): the canonical property. This is where you watch indexing, performance, and search traffic.
- `campdayfunk.com` (URL prefix property): exists only to monitor that redirects keep working. All URLs in this property should classify as "Page with redirect," which is the desired state.

Verification method: DNS TXT records (already in place in both zones).

### Sitemap

Submitted at `https://dayfunk.com/sitemap.xml` under the dayfunk.com property.

### Expected "issues" that are NOT problems

Both properties will show "Page with redirect" pages under Page Indexing. These are correct and permanent:

- dayfunk.com property: `http://dayfunk.com/` and `http://www.dayfunk.com/` redirect to https. They will always show here.
- campdayfunk.com property: all four URL variants (http/https × apex/www) redirect to dayfunk.com. They will always show here.

Do NOT click "Validate Fix" on these. Validation will always fail because the redirects are intentional and permanent. Google's emails about these are informational, not actionable.

Note: "Validate Fix" was clicked on `http://dayfunk.com/` in May 2026. Validation ran 5/13–5/18 and failed as expected. This did not affect SEO. The failed validation state persists in Search Console indefinitely but is harmless — do not start a new validation.

### Issues that ARE worth acting on

If Search Console flags any of the following for dayfunk.com URLs, investigate:

- Server error (5xx)
- Submitted URL not found (404) for URLs in the sitemap
- Soft 404
- Excluded by 'noindex' tag (unintentional)
- Canonical conflict where Google chose a non-dayfunk.com canonical (this was the original problem, now resolved)

## Structured data

The homepage includes JSON-LD for Event and MusicGroup schema. Search Console's Events enhancement report should show valid items detected. When events are added or removed from the lineup in `index.html`, re-verify the Events report after the next crawl.

## History of canonical issue (resolved May 2026)

For context if a similar issue arises: Google was choosing campdayfunk.com as the canonical over dayfunk.com because campdayfunk had been the original domain. Fix was a 301 redirect rule in the campdayfunk.com Cloudflare zone catching all traffic and sending it to dayfunk.com. Resolution took roughly a week after the redirect rule deployed and re-indexing was requested. The dayfunk.com URL inspection now shows "URL is on Google" with green checks across the board.

## Common change scenarios

### Adding a new subdomain or domain
1. Add DNS record in Cloudflare with appropriate proxy state
2. If it should serve the site, add as Pages custom domain
3. If it should redirect elsewhere, create a redirect rule in that zone (not in dayfunk.com zone)
4. Add as a property in Search Console if you want to monitor it

### Editing site content
1. Edit files in the `craigspree/dayfunk-website` GitHub repo
2. Commit to `main`
3. Cloudflare Pages auto-deploys within 1-2 minutes
4. No cache purge needed unless changes don't appear after 5 minutes

### Investigating a redirect not firing
1. Confirm DNS record is proxied (orange cloud) in the relevant zone
2. Confirm the redirect rule lives in the same zone as the incoming hostname
3. Confirm rule order (more specific rules above the catch-all)
4. Confirm the domain is NOT attached as a Pages custom domain (Pages will intercept before the rule fires)

## Contacts

- Site owner: Nick Gunz (event brand owner, not technical)
- Infrastructure owner: Craig Paulnock
- GitHub account for repo: `craigspree`
