---
name: lead-scraper-creator
description: >
  Master index and entry point for the bespoke lead-generation agent system in this repo.
  Trigger on: "the lead scraper creator", "the workshop", "the lead workshop", "build me a
  lead scraper", "set up lead gen for [client]", "Mountain Falls-style lead gen", "new lead-gen
  client", or any request to design, build, or spec a bespoke lead-generation/scraping agent
  for a client. Treat this as the front door: on trigger, load the lead-scraping-agent skill
  (the full five-layer build guide) and pull in its reference docs as needed. Also indexes the
  complementary installed skills (gmaps-leads, apollo-lead-finder, icp-identification,
  scraperapi-*, etc.) by pipeline layer, and the client-intake wizard prototype.
---

# Lead Scraper Creator — System Index

One name to remember instead of seven file paths. This skill is the map of the entire
lead-generation build system in this repo; everything below is a pointer, not a duplicate —
the actual build guidance lives in `lead-scraping-agent` and its references.

**When this triggers:** read `.claude/skills/lead-scraping-agent/SKILL.md` next. That file is
the full build guide; this one just tells you where everything is.

---

## The Architecture in Five Lines

Every client build is the same pipeline with different configuration:

1. **Source** — pick discovery sources from the ICP (Google Maps, LinkedIn-via-proxy, job boards, directories, SERPs). One primary, one fallback.
2. **Extraction** — normalize raw source output into one flat JSON record per lead. Dumb on purpose.
3. **Enrichment + Scoring** — firmographic APIs fill the record; Claude scores ICP fit 1–10 and writes the `recommended_angle` outreach hook — the one generative output in an otherwise deterministic pipeline.
4. **Self-Healing** — error classification, retry with backoff, DOM-change auto-remediation, schema gates, daily health checks. What makes it a product instead of a script.
5. **Delivery** — CRM push, gated outbound injection, dashboard, or CSV fallback. The only layer the client touches.

Full guide: `.claude/skills/lead-scraping-agent/SKILL.md` — including packaging/pricing and the
ten-step build order.

---

## Reference Docs (deep detail, one per concern)

All under `.claude/skills/lead-scraping-agent/references/`:

- `sources.md` — per-source setup, rate limits, ToS risk, the never-scrape-LinkedIn-directly rule, primary+fallback selection table.
- `extraction.md` — Apify actor configs, Playwright selector strategy, the n8n normalization flow, email validation gating.
- `enrichment.md` — Clearbit/Hunter setup, the ICP scoring prompt, intent-signal detection, dedup logic, scoring calibration rubric.
- `self-healing.md` — n8n blueprints for failure detection, backoff retries, LLM selector remediation with a confidence gate, schema drift, daily health checks.
- `delivery.md` — HubSpot/Pipedrive/Airtable field mappings, Smartlead/Instantly injection with score + opt-out + suppression gating, dashboard widgets, CSV fallback.
- `storage-google-sheets.md` — the default backend (see next section).

---

## The Storage Decision

**Google Sheets is the default backend for every new client** — one workbook per client
(`{client_name} — Lead Pipeline`, five tabs: Leads, Suppression List, Client Config, Scraping
Errors, Source Health). **Supabase is the scale path**, migrated to when row volume,
write concurrency, or dashboard needs demand it. Column names are identical across both
backends by design, so migration is copy-rows, not redesign. Everything — tab layout,
Sheets-native dedup, quota batching, migration triggers — is in
`references/storage-google-sheets.md`.

---

## Client Intake

`prototypes/intake-wizard.html` — the client-intake wizard prototype ("Lead Workshop — Bespoke
Agent Build Console"). Walks through client name → ICP definition → source selection →
enrichment config → delivery config, and generates the per-client build spec whose fields map
1:1 onto the `Client Config` tab in `storage-google-sheets.md`.

---

## Complementary Installed Skills, by Pipeline Layer

These are vendored third-party skills already installed in this repo (tracked in
`skills-lock.json` — don't edit them). Reach for them instead of building a layer from scratch
when one fits:

**Source / Extraction (Layers 1–2)**
- `gmaps-leads` — Google Maps B2B scraping with website enrichment and contact extraction.
- `scrape-leads` — Apify-based lead scraping + verification + Google Sheets output, end to end.
- `job-scraper` — LinkedIn/Indeed job postings; the hiring-signal source for Layer 3c intent detection.
- `classify-leads` — LLM classification for distinctions scraping can't make (e.g. product SaaS vs. agency).
- `casualize-names` — converts formal names/companies/cities to casual forms for cold-email merge fields.

**Enrichment / ICP (Layer 3)**
- `icp-identification` — research a company or idea and define the ICP; the upstream step before any build.
- `comprehensive-enrichment` — enrich any person/company from any identifier (email, domain, LinkedIn URL, handle).
- `gtm-enrichment-smart` — multi-provider waterfall enrichment, cheap APIs first, ~$0.04–0.10/lead with confidence scores.
- `inbound-lead-enrichment` — fills gaps on inbound leads: role, seniority, stakeholders, CRM cross-check.

**Prospecting / Contacts**
- `apollo-lead-finder` — two-phase Apollo.io prospecting: free search, then selective credit-spend enrichment.
- `company-contact-finder` — decision-makers at a named company via Apollo/Crustdata/Fiber/PDL.
- `targeted-prospecting` — account lists with decision-makers, verified contacts, and hiring/intent signals.

**Qualification / Dedup / Signals**
- `lead-qualification` — conversational criteria intake → reusable qualification prompt → batch scoring with verdicts.
- `contact-cache` — CSV-backed contact database, dedup by LinkedIn URL or email, prevents duplicate outreach.
- `signal-scanner` — buying-signal detection across TAM/watchlist (headcount, funding, tech-stack, hiring diffs).

**ScraperAPI Suite (infrastructure for Layers 1–2)**
- `scraperapi-agent-onboarding` — start here for anything ScraperAPI: setup, product choice, API keys.
- `scraperapi-crawler` — whole-site/section crawling by following links.
- `scraperapi-datapipeline` — managed scheduled scraping projects with webhook delivery.
- `scraperapi-lead-enrichment` — contact-card building from any seed via ScraperAPI search + fetch.
- `scraperapi-mcp` — reference for the 22 ScraperAPI MCP tools (Google, Amazon, Walmart, eBay, Redfin, crawl).
- `scraperapi-n8n` — generates n8n workflows using the official ScraperAPI community node.
- `scraperapi-scraper-builder` — produces complete runnable Python/Node scraper scripts on ScraperAPI.
