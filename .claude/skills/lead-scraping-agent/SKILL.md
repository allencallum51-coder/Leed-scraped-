---
name: lead-scraping-agent
description: >
  Use this skill whenever building, designing, architecting, or modifying a lead scraping agent
  or workflow. Covers source selection, extraction logic, enrichment pipelines, self-healing
  strategies, deduplication, CRM delivery, and packaging the output as a sellable product.
  Trigger on any mention of: lead generation, lead scraping, prospect sourcing, outbound data,
  contact enrichment, ICP targeting, lead pipeline, or building an agent that finds/qualifies
  companies or contacts. Always use this skill — even if the user only mentions one component
  (e.g. "just the enrichment step") — because the architecture decisions are interdependent.
---

# Lead Scraping Agent — Build Skill

This is a build guide, not a script. It describes how to construct a bespoke lead scraping
agent for any client, niche, or ICP — the same five-layer architecture every time, with the
client-specific decisions (sources, ICP definition, delivery target) treated as configuration
rather than rewrites. Nothing here is tied to a single brand or vertical.

The reason it's worth reading end-to-end even when you only need one piece: the layers depend
on each other in non-obvious ways. Delivery gating reads scores produced in enrichment;
self-healing replays payloads captured in extraction; enrichment prompts read config written
during source setup. Build one layer in isolation and you'll usually get its interface wrong.

---

## Architecture Overview

Five layers, built and validated in order. Skipping ahead creates debt that surfaces later as
broken self-healing logic — the layer least fun to debug.

```
[1. Source Layer]
       ↓
[2. Extraction Layer]
       ↓
[3. Enrichment + Scoring Layer]  ← LLM lives here
       ↓
[4. Self-Healing Layer]          ← runs in parallel / on failure
       ↓
[5. Delivery Layer]
       ↓
[Dashboard / CRM / Webhook]
```

Orchestration (n8n / Inngest) wraps all five. Keep each layer a discrete n8n workflow section
or sub-workflow — the point is that any one of them can be tested, swapped, or healed without
touching the others.

**Storage default — read this before touching any schema below.** Every table in this guide
(`leads_raw`, `leads_enriched`, `campaign_config`, `scraping_errors`, `source_health`) is
written as a Postgres/Supabase schema, because Supabase is the eventual scale target. But the
*default* backing store for a new client is a Google Sheet, not Supabase. See
`references/storage-google-sheets.md` for the sheet/tab layout (including outreach-status and
opt-out tracking columns) and the criteria for migrating a client from Sheets to Supabase once
volume or concurrency demands it. Column names are deliberately identical across both backends,
so migration is a lift-and-shift, not a redesign.

---

## Layer 1 — Source Selection

Sources follow from the ICP, not the other way around. Resist the urge to integrate everything
on day one: start with one primary and one fallback source, validate the pair end-to-end, and
only then expand.

| Source Type         | Best For                          | Tool / Method                        | Risk Level |
|---------------------|-----------------------------------|--------------------------------------|------------|
| Google Maps/Places  | Local businesses, SMBs            | Google Places API (official)         | Low        |
| LinkedIn            | B2B, role-based targeting         | PhantomBuster, Bright Data, Apify    | High       |
| Job boards          | Companies with hiring signals     | Apify actors, custom scraper         | Medium     |
| Industry directories| Niche verticals                   | Custom scraper + Cheerio/Playwright  | Medium     |
| Company websites    | Tech stack, size, intent signals  | Hunter.io, Clearbit, BuiltWith API   | Low        |
| Google Search (SERPs)| Broad discovery                  | SerpAPI, ValueSERP                   | Low        |

**The LinkedIn rule is absolute:** never scrape LinkedIn directly. Always go through a
compliant proxy layer (PhantomBuster cloud, Bright Data's LinkedIn dataset, or Apify's LinkedIn
actors). Raw scraping earns permanent IP bans within hours and creates legal exposure under
LinkedIn's ToS — there is no volume of leads worth that.

→ See `references/sources.md` for per-source setup guides and rate limits.

---

## Layer 2 — Extraction

Extraction has exactly one job: turn a source's raw response into a clean, flat JSON object
per lead. No enrichment, no scoring, no API calls beyond the source itself. Keeping this layer
dumb is what makes it swappable when a source breaks.

**Minimum required fields per raw lead:**
```json
{
  "company_name": "",
  "website": "",
  "industry": "",
  "location": "",
  "employee_count_range": "",
  "source": "",
  "scraped_at": "",
  "raw_contact_name": "",
  "raw_contact_role": "",
  "raw_email": "",
  "raw_linkedin_url": ""
}
```

**n8n node pattern:**
- HTTP Request node → source API / scraper endpoint
- Function node → normalize to the flat schema above
- Set node → tag `source` and `scraped_at`
- Write to Supabase `leads_raw` table

**Email extraction:** scraped emails are guesses until proven otherwise. Pass every one through
a validation API (ZeroBounce or NeverBounce) before enrichment. Invalid or risky addresses cost
twice — once as noise in the pipeline, and again as deliverability damage if leads feed into
outbound sequences.

→ See `references/extraction.md` for Apify actor configs, Playwright selectors, and rate limiting.

---

## Layer 3 — Enrichment + Scoring

Everything up to this point is plumbing that any scraper could do. Layer 3 is where the
pipeline earns its price: the LLM — Claude (`claude-sonnet-4-6`) — turns a flat contact record
into a qualified, contextualised lead with a score and a reason to reach out.

Claude powers two outputs here: the ICP fit score (a constrained classification task) and the
`recommended_angle` field (the one genuinely generative output in an otherwise deterministic
pipeline — see 3b).

### Enrichment steps (in order):

**3a. Company enrichment**
Call Clearbit Enrichment API or Hunter.io Company API to fill in missing firmographics
(headcount, revenue range, tech stack, social profiles). Scraped estimates are unreliable;
grounding the record in a real firmographic dataset is what makes the ICP score in 3b
trustworthy.

**3b. ICP fit scoring (LLM)**
Pass the enriched company record to Claude with a structured system prompt:

```
You are an ICP qualification agent. Given a company profile, score it 1–10 for fit against
the following Ideal Customer Profile:

ICP: {{icp_definition}}

Return ONLY valid JSON:
{
  "icp_score": <int 1-10>,
  "fit_reasons": ["<reason>", ...],
  "disqualifiers": ["<reason>", ...],
  "recommended_angle": "<one sentence personalised outreach hook>"
}
```

Note the asymmetry in this prompt: `icp_score`, `fit_reasons`, and `disqualifiers` are
extractive — they should follow mechanically from the profile and the ICP definition. But
`recommended_angle` asks the model to make a small creative leap: connect something specific
about this company to a reason they'd take a call. That's the field to watch in QA — the JSON
can validate, the score can look reasonable, and the angle can still be a generic "I noticed
your company is growing" that could apply to anyone. A good angle cites the same specific
evidence that drove the score; a lazy one is the tell that the prompt needs tightening.

Store `icp_definition` in a Supabase config table so it can be updated per campaign without
redeploying the workflow.

**3c. Intent signal detection (optional, high-value)**
If the source was a job board, extract hiring signals:
- Role being hired → infer pain point (e.g. "hiring Sales Ops" = scaling sales motion)
- Job description keywords → pass to Claude to infer the business problem

**3d. Deduplication**
Before writing to the enriched table, check for existing records:
```sql
SELECT id FROM leads_enriched
WHERE website = $1 OR (company_name = $2 AND location = $3)
LIMIT 1;
```
If match found → update `last_seen_at` and skip. Do not create duplicate rows.

→ See `references/enrichment.md` for prompt templates and Clearbit/Hunter API setup.

---

## Layer 4 — Self-Healing

A scraper that works today is a scraper that will break — DOMs change, APIs revise their
schemas, anti-bot measures rotate. Self-healing is the difference between a product and a
one-off script: the agent detects, logs, adapts, and alerts, so a failure becomes a log entry
instead of a support call.

### Self-Healing Strategies

#### 4a. Extraction failure detection
In the extraction n8n workflow, wrap every HTTP Request / Apify actor call in an error branch:

```
[HTTP Request] → [Success branch] → continue pipeline
              ↘ [Error branch]   → log to `scraping_errors` table
                                 → trigger self-heal sub-workflow
```

The `scraping_errors` table schema:
```sql
CREATE TABLE scraping_errors (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  source TEXT,
  error_type TEXT,        -- 'rate_limit' | 'dom_change' | 'auth_block' | 'timeout'
  error_message TEXT,
  payload JSONB,          -- the original request payload
  retry_count INT DEFAULT 0,
  resolved BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

#### 4b. Retry with backoff
On rate limit or timeout errors, schedule an automatic retry using n8n's Wait node:
- Retry 1: wait 5 minutes
- Retry 2: wait 30 minutes
- Retry 3: wait 2 hours
- After 3 failures: mark `resolved = false`, alert via Slack/email webhook

#### 4c. DOM change detection (for HTML scrapers)
After each run, store a hash of the target page's critical selector output. On next run:
1. Extract the selector
2. Compare hash
3. If changed → flag as `dom_change`, pause that source, alert

Use an LLM to attempt auto-remediation:
```
The following CSS selector no longer returns expected data from {url}.
Previous working selector: {old_selector}
Here is the current page HTML (truncated): {html_snippet}
Suggest the updated CSS selector that targets the equivalent element.
Return ONLY the new selector string.
```
Write the suggested selector back to a `scraper_config` table and resume. Log as requiring
human verification if confidence is low (i.e. LLM response contains uncertainty language).

#### 4d. Proxy rotation
If using residential or datacenter proxies (Bright Data, Oxylabs), configure rotation at the
proxy provider level. In n8n, store proxy credentials in environment variables, not hardcoded
in nodes. Rotate API keys on a schedule using a separate n8n maintenance workflow.

#### 4e. Schema drift detection (for API sources)
After enrichment, validate every outbound record against a Zod or JSON Schema definition
before writing to `leads_enriched`. If required fields are missing or types have changed,
log to `schema_errors` and halt that record — do not write partial data.

#### 4f. Health check workflow
Create a dedicated n8n workflow that runs daily:
1. Runs a single test scrape against each configured source (with a known test target)
2. Validates the output matches expected schema
3. Writes pass/fail to a `source_health` table
4. Posts a daily health summary to a Slack channel or email

→ See `references/self-healing.md` for n8n node blueprints for each strategy above.

---

## Layer 5 — Delivery

Nobody buys a database. Delivery is the layer that turns pipeline output into something the
client's sales team actually uses — the client should never have to touch raw data.

### Delivery options (choose based on client context):

**5a. CRM push (preferred)**
Direct webhook or API integration into the client's CRM:
- HubSpot: use HubSpot API node (Create Contact + Create Company)
- Pipedrive: Pipedrive API node (Create Person + Create Organization)
- Airtable: Airtable node (append to base)

Map enriched fields to CRM fields. Include `icp_score` and `recommended_angle` as custom
properties so the sales team sees qualification context inline with the lead record.

**5b. Outbound sequence injection**
Push directly into a sending tool:
- Smartlead / Instantly: use their API to add contacts to active campaigns
- Only inject leads with `icp_score >= 7` to protect sender reputation

**5c. Dashboard (see Packaging section)**
Build a lightweight Next.js + Supabase dashboard surfacing:
- Lead volume over time
- Source breakdown
- ICP score distribution
- Enrichment coverage %
- Self-healing alerts

**5d. Scheduled export (fallback)**
If CRM integration isn't possible, generate a weekly CSV export via n8n's Spreadsheet node
and deliver via email. This is the lowest-value delivery option — use only as a fallback.

---

## Packaging as a Sellable Product

A list of leads is a commodity; the pipeline that keeps producing them is not. Every component
below exists to move the engagement from "bought a list once" to "pays a retainer":

| Component | What It Provides | Why It Matters for Pricing |
|-----------|-----------------|---------------------------|
| Continuous pipeline | New leads weekly/daily, not a one-time list | Justifies subscription over one-off fee |
| ICP scoring | Only qualified leads reach the client | Replaces manual SDR qualification time |
| Recommended angle | Personalised outreach hook per lead | Saves copywriting time per contact |
| CRM integration | Leads land in existing workflow | Zero friction for sales team |
| Self-healing | Pipeline runs without manual fixes | Justifies retainer (you're not babysitting) |
| Dashboard | Visible ROI, volume, quality metrics | Supports renewal conversations |

### Pricing model (suggested)
- **Setup fee:** One-time fee covering source config, ICP definition, CRM mapping, and
  self-healing setup. Scope based on number of sources and CRM complexity.
- **Monthly retainer:** Covers pipeline runs, proxy/API pass-throughs, self-healing
  monitoring, and a monthly health review. Price by lead volume tier.
- **Overage:** Per-lead fee above the committed monthly volume.

Do not sell per-lead-only without a monthly minimum. Scraper yield fluctuates when sources
break, and a pure per-lead price makes every breakage a revenue event for you and a billing
dispute for the client.

---

## Build Order (Recommended Sequence)

1. Define ICP with client → store in `campaign_config` Supabase table
2. Select and configure primary source → test extraction, validate raw schema
3. Set up email validation → connect ZeroBounce / NeverBounce
4. Build enrichment workflow → test Claude scoring against 20 known leads
5. Build self-healing layer → test failure injection, verify retry + alert logic
6. Build delivery integration → test CRM push end-to-end with dummy data
7. Set up health check workflow → confirm daily runs and alerting
8. Build dashboard (if in scope)
9. Run full pipeline against real target set → review ICP score distribution
10. Hand off to client with documented ICP definition and change request process

---

## Reference Files

- `references/sources.md` — Per-source setup, rate limits, ToS risk levels, recommended tools
- `references/extraction.md` — Apify configs, Playwright selectors, n8n node patterns
- `references/enrichment.md` — Claude prompt templates, Clearbit/Hunter setup, scoring rubrics
- `references/self-healing.md` — n8n blueprints for retry, DOM healing, health checks
- `references/delivery.md` — CRM field mappings, Smartlead/Instantly API patterns, dashboard schema
- `references/storage-google-sheets.md` — Default Google Sheets backend: workbook/tab layout, Sheets-native dedup and suppression, Supabase migration criteria

---

## Key Constraints

- **Never hardcode API keys** in n8n nodes — always use n8n credentials store or environment variables
- **Never write unenriched leads to the delivery table** — keep `leads_raw` and `leads_enriched` separate
- **Never scrape LinkedIn directly** — always use a compliant proxy layer
- **Always validate emails** before enrichment runs — saves LLM tokens on invalid records
- **Always define the ICP in a config table**, not in the prompt string — makes it updatable without redeployment
