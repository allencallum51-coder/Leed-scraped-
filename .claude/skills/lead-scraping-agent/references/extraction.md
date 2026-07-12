# Extraction Layer — Apify Configs, Playwright Selectors, n8n Patterns

Reference for Layer 2. The discipline of this layer is what it *doesn't* do: no scoring, no
enrichment API calls, no judgment. It turns a source's raw response into the flat schema defined
in the main SKILL.md, and stops. Everything clever belongs in Layer 3, where it can be changed
without re-scraping anything.

---

## Apify Actor Configuration

**Choosing an actor:**
- Prefer an existing well-maintained actor over building a custom one — check run count and last
  update date in the Apify Store before committing. An actor with thousands of recent runs is an
  actor someone else is already keeping alive when the target site changes.
- For LinkedIn, job boards, and most e-commerce/directory sites, an official or community actor
  almost always exists.

**Standard actor input pattern (example: LinkedIn Sales Navigator search actor):**
```json
{
  "searchUrl": "https://www.linkedin.com/sales/search/people?query=...",
  "maxResults": 200,
  "proxyConfiguration": {
    "useApifyProxy": true,
    "apifyProxyGroups": ["RESIDENTIAL"]
  }
}
```

**Running via n8n:**
- HTTP Request node → `POST https://api.apify.com/v2/acts/{actor_id}/runs?token={API_TOKEN}`
  with the input JSON as the body.
- Poll `GET /v2/actor-runs/{run_id}` until `status` is `SUCCEEDED` — or better, configure a
  webhook on the actor run (`webhooks` param) that POSTs back to an n8n Webhook node on
  completion, and skip polling entirely.
- Fetch results: `GET https://api.apify.com/v2/datasets/{dataset_id}/items`.

**Cost control:** Set `maxResults` conservatively per run and track `computeUnits` consumed per
run in the `scraping_errors`-adjacent metrics table (see self-healing.md) so cost-per-lead is
visible before scaling a source. You can't price a retainer sensibly without this number.

---

## Playwright (Custom Scrapers)

A full browser is the tool of last resort, not the default. Use Playwright only when:
- The target renders content client-side (React/Vue SPA with no server-rendered fallback), or
- The flow requires interaction (search form submission, pagination via "load more" clicks,
  login-gated content).

For everything else, prefer a lighter HTTP + Cheerio/BeautifulSoup approach — it's faster,
cheaper, and has far fewer moving parts to break.

**Selector strategy:**
- Prefer stable attributes over generated class names: `data-testid`, `id`, `aria-label`, or
  semantic tags (`<article>`, `<table>`) over `.css-x7y2z1` style hashed classes that change on
  every deploy.
- Scope selectors to the narrowest reliable container first, then extract fields relative to it,
  rather than one giant page-wide selector list. The payoff comes at breakage time: a DOM change
  takes out one field instead of the whole record, and Layer 4's remediation has a much smaller
  problem to solve.

```js
// Example: extracting a listing card
const cards = await page.$$('[data-testid="listing-card"]');
for (const card of cards) {
  const name = await card.$eval('h3', el => el.textContent.trim());
  const website = await card.$eval('a[href^="http"]', el => el.href).catch(() => null);
}
```

**Rendering settings:**
- Run headless with a realistic user agent and viewport; set `waitUntil: 'networkidle'` only when
  necessary — it's slow. Prefer waiting on the specific selector you need
  (`page.waitForSelector`) over a blanket network-idle wait.
- Route through a residential/datacenter proxy at the browser context level
  (`browser.newContext({ proxy: {...} })`) rather than per-request, so cookies and session state
  stay consistent with the assigned IP — a session whose IP changes mid-flow looks exactly like
  the bot it is.

---

## n8n Node Pattern (Standard Extraction Flow)

```
[Schedule Trigger / Webhook]
        ↓
[HTTP Request] → source API, or [Apify: Run Actor + Get Dataset Items]
        ↓
[Function node] → normalize response to flat schema:
                   { company_name, website, industry, location,
                     employee_count_range, source, scraped_at,
                     raw_contact_name, raw_contact_role,
                     raw_email, raw_linkedin_url }
        ↓
[Set node] → tag `source` = static string, `scraped_at` = {{ $now }}
        ↓
[Supabase node] → insert into `leads_raw`
        ↓
[Error branch on any node above] → see self-healing.md 4a
```

**Function node normalization example:**
```js
return items.map(item => ({
  json: {
    company_name: item.companyName ?? item.name ?? null,
    website: item.website ?? item.companyUrl ?? null,
    industry: item.industry ?? null,
    location: item.location ?? item.city ?? null,
    employee_count_range: item.employeeCountRange ?? null,
    source: 'linkedin_sales_nav',
    scraped_at: new Date().toISOString(),
    raw_contact_name: item.fullName ?? null,
    raw_contact_role: item.title ?? null,
    raw_email: item.email ?? null,
    raw_linkedin_url: item.profileUrl ?? null,
  }
}));
```

Note the `?? null` on every field: a missing value should arrive downstream as an explicit
null, never as `undefined` or an absent key — Layer 4's schema validation (self-healing.md 4e)
depends on the shape being consistent.

---

## Rate Limiting

- The source's own documented limits win (see sources.md per-source table).
- For custom scrapers with no documented limit, default to 1 request/second per domain and back
  off exponentially on any 429/503 response before retrying (see self-healing.md 4b for the
  n8n-level retry schedule).
- Batch Apify actor runs rather than firing one run per lead — most actors accept an array of
  search queries/URLs in a single run, which is both cheaper and easier to rate-limit centrally.

---

## Email Extraction & Validation

Never write a scraped email straight to `leads_enriched`. Pipe every `raw_email` through
ZeroBounce or NeverBounce before it reaches Layer 3:

```
[Supabase: read unvalidated leads_raw]
        ↓
[HTTP Request] → ZeroBounce validate endpoint (per email)
        ↓
[Switch node] → status == 'valid' → continue to enrichment
              → status in ('invalid','spamtrap','abuse') → mark raw_email null, log, continue
              → status == 'catch-all'/'unknown' → flag low_confidence_email = true, continue
```

Two things pay for this step: LLM tokens (no point scoring a lead whose contact route is dead)
and sender reputation (one spamtrap in an outbound sequence damages deliverability for every
future send). Note that invalid emails null the field and *continue* — a company can still be a
good lead with a bad email; the record just can't be injected into outbound until a valid
contact route exists.
