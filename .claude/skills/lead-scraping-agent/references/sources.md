# Source Layer — Setup Guides & Rate Limits

Per-source configuration reference for Layer 1. Pick one primary and one fallback source per
campaign — do not integrate every row on day one.

---

## Google Maps / Places API

**Best for:** Local businesses, SMBs, service contractors, anything with a physical location.

**Setup:**
1. Enable the Places API in Google Cloud Console, generate an API key, restrict it to Places API
   only (IP or referrer restriction in production).
2. Use the `places:searchText` or `places:searchNearby` endpoint with a category + geography query
   (e.g. "roofing contractors in Austin, TX").
3. Follow pagination via `nextPageToken` — each page returns up to 20 results, max 60 per query
   before you need to narrow the geography further.
4. Call Place Details for each result to get phone, website, and hours (costs an extra request
   per place — budget accordingly).

**Rate limits:** 6,000 requests/minute default quota (soft, raisable). Real constraint is cost:
Text Search ~$32/1000 requests, Place Details ~$17/1000 requests (Essentials SKU as of last
pricing check — verify current pricing before quoting a client).

**ToS risk:** Low — official API, fully compliant.

---

## LinkedIn (via compliant proxy layer only)

**Best for:** B2B, role-based targeting, decision-maker discovery.

**Never scrape LinkedIn directly** — no raw HTTP requests, no headless browser hitting
linkedin.com directly. Use one of:

- **PhantomBuster** — "Phantoms" for Sales Navigator search export, profile scraping, post
  engagement scraping. Runs in PhantomBuster's cloud, rotates through their IP pool.
- **Bright Data LinkedIn dataset** — pre-scraped, continuously refreshed dataset accessed via
  API; avoids live scraping entirely for profile/company data.
- **Apify LinkedIn actors** — `apify/linkedin-jobs-scraper`, `apify/linkedin-sales-navigator`,
  etc. Apify handles proxy rotation and CAPTCHA solving internally.

**Setup (PhantomBuster example):**
1. Connect a LinkedIn session cookie (a dedicated "scraper" seat, never a real employee's
   personal account) to PhantomBuster.
2. Configure the Phantom with a Sales Navigator search URL or a list of profile URLs.
3. Schedule via PhantomBuster's own scheduler or trigger via their API from n8n.
4. Output lands in PhantomBuster's result store — pull via API into the n8n workflow.

**Rate limits:** Self-imposed — cap searches to what a real human user could plausibly do
(roughly 80-100 profile views/day per seat). Exceeding this is what triggers bans regardless of
which proxy layer you use.

**ToS risk:** High even through a proxy layer — LinkedIn's ToS prohibits automated data
collection outright. Compliant proxy tools reduce ban risk and legal exposure vs. raw scraping,
but do not eliminate it. Flag this risk explicitly to clients before building on LinkedIn as a
primary source.

---

## Job Boards

**Best for:** Companies with active hiring signals (a strong intent proxy).

**Setup:**
- Apify actors exist for Indeed, LinkedIn Jobs, and Glassdoor (`apify/indeed-scraper`, etc.).
- For a custom scraper: most job boards render listings server-side, so Cheerio (Node) or
  BeautifulSoup (Python) against the search results HTML is usually sufficient — reserve
  Playwright for boards that lazy-load via JS.
- Query by role/title relevant to the client's ICP pain point (e.g. "Sales Ops Manager" implies
  a company scaling its sales motion).

**Rate limits:** No official API for most boards; self-throttle to 1 request per 2-4 seconds per
domain and rotate user agents / IPs via the Apify proxy pool to avoid soft blocks.

**ToS risk:** Medium — most job boards' ToS discourage scraping but enforcement is inconsistent
compared to LinkedIn. Prefer sources with an official API/RSS feed (e.g. some ATS platforms
expose public job feeds) over raw HTML scraping where available.

---

## Industry Directories

**Best for:** Niche verticals not well covered by Google Maps or LinkedIn (e.g. licensed
professionals, trade associations, chamber-of-commerce member lists).

**Setup:**
- Custom scraper with Cheerio for static directory pages; Playwright only if the directory uses
  client-side rendering or requires interaction (search forms, "load more" buttons).
- Directories are typically small (hundreds to low thousands of entries) — a single scheduled
  run per week is usually sufficient rather than continuous polling.

**Rate limits:** No standard — inspect `robots.txt` first and throttle to a conservative 1
request/second unless the site states otherwise.

**ToS risk:** Medium — varies widely by directory; check terms before building, especially for
directories run by professional licensing bodies (some explicitly prohibit bulk collection).

---

## Company Websites (Tech Stack, Size, Intent Signals)

**Best for:** Enriching a company record once you already have a domain, or confirming firmo-
graphics scraped from another source.

**Setup:**
- **Hunter.io** — Domain Search endpoint returns known email patterns and public email
  addresses for a domain. Company Search returns firmographic metadata.
- **Clearbit Enrichment API** — richest firmographic dataset (headcount, revenue range, tech
  stack, funding, social profiles) keyed by domain or email.
- **BuiltWith API** — tech stack detection specifically; useful for ICP filters like "uses
  HubSpot" or "runs on Shopify."

**Rate limits:** Governed by your plan tier on each provider — Hunter and Clearbit both meter by
monthly lookup credits, not requests/second. Budget credits per campaign volume before scaling.

**ToS risk:** Low — official APIs, designed for this use case.

---

## Google Search (SERPs)

**Best for:** Broad discovery when you don't have a clean source list yet — "find companies that
mention X on their homepage," competitor customer lists, press mentions.

**Setup:**
- **SerpAPI** or **ValueSERP** — pass a search query, get back structured SERP results
  (organic listings, snippets, sometimes knowledge panels) without violating Google's ToS on
  direct scraping.
- Useful query patterns: `site:linkedin.com/company "hiring" [role]`, `intitle:"case study"
  [competitor name]`, `"powered by" [target software]`.

**Rate limits:** Metered by plan (requests/month). SerpAPI free tier is low-volume (~100/month);
budget a paid tier for production use.

**ToS risk:** Low — these are third-party APIs built specifically to provide SERP data legally,
not raw scrapers hitting google.com.

---

## Choosing Primary + Fallback

| Client Profile | Primary Source | Fallback Source |
|---|---|---|
| Local service business (SMB) | Google Maps/Places | Industry directory |
| B2B SaaS targeting specific roles | LinkedIn (via proxy) | Job boards (hiring signal) |
| Enterprise / niche vertical | Industry directory | Google Search (SERPs) |
| Any — enrichment-only, no discovery | Company websites (Hunter/Clearbit) | N/A |

Expand to a third source only after the primary + fallback pair is validated end-to-end
(Layer 2 schema conformance, Layer 4 health checks passing for at least one week).
