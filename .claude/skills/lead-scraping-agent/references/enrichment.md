# Enrichment Layer — Prompt Templates, API Setup, Scoring Rubrics

Reference for Layer 3. This is where raw scraped rows become qualified, contextualized leads.

---

## 3a. Company Enrichment API Setup

**Clearbit Enrichment API:**
- Endpoint: `GET https://company.clearbit.com/v2/companies/find?domain={domain}`
- Auth: Bearer token (secret API key) — store in n8n credentials, never in the node.
- Returns: headcount range, estimated revenue, tech stack, category/industry tags, social
  profiles, funding data (if available).
- Fallback behavior: if `domain` lookup returns 404, retry with `name` search endpoint
  (`/v1/companies/search?query={company_name}`) before giving up.

**Hunter.io Company API:**
- Endpoint: `GET https://api.hunter.io/v2/companies/find?domain={domain}&api_key={key}`
- Cheaper alternative/complement to Clearbit; strongest for email pattern + headcount data,
  weaker on funding/tech stack than Clearbit.

**n8n pattern:**
```
[Supabase: read leads_raw where not yet enriched]
        ↓
[HTTP Request] → Clearbit (or Hunter) company lookup by domain
        ↓
[Merge node] → combine raw lead fields + enrichment response
        ↓
[continue to 3b]
```

Cache enrichment responses in a `company_enrichment_cache` table keyed by domain with a 30-day
TTL — company firmographics don't change fast enough to justify a fresh API call per lead when
multiple contacts share the same company.

---

## 3b. ICP Fit Scoring (LLM)

**System prompt template:**
```
You are an ICP qualification agent. Given a company profile, score it 1–10 for fit against
the following Ideal Customer Profile:

ICP: {{icp_definition}}

Company profile:
{{enriched_company_json}}

Return ONLY valid JSON:
{
  "icp_score": <int 1-10>,
  "fit_reasons": ["<reason>", ...],
  "disqualifiers": ["<reason>", ...],
  "recommended_angle": "<one sentence personalised outreach hook>"
}
```

**Model call settings:**
- Use a low temperature (0–0.3) — this is a classification/scoring task, not creative writing;
  consistency across runs matters more than variety.
- Enforce structured output (JSON mode / tool-use schema) rather than parsing free text — avoids
  a whole class of self-healing cases caused by malformed LLM output.
- Validate the response against a schema before writing to `leads_enriched`; on schema failure,
  retry once with a "your last response was invalid JSON, return only the JSON object" follow-up
  before logging to `schema_errors` (see self-healing.md 4e).

**`icp_definition` storage:**
```sql
CREATE TABLE campaign_config (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  campaign_name TEXT UNIQUE NOT NULL,
  icp_definition TEXT NOT NULL,   -- freeform description fed into the prompt
  min_icp_score INT DEFAULT 6,    -- threshold for downstream delivery/outbound injection
  updated_at TIMESTAMPTZ DEFAULT now()
);
```
Pull `icp_definition` into the prompt at runtime via a Supabase lookup node — never hardcode ICP
criteria into the prompt string itself.

---

## 3c. Intent Signal Detection

When the source is a job board, pass the job posting text through a lighter-weight prompt before
the main ICP scoring call:

```
Given this job posting, infer the business problem the hiring company is likely trying to solve.

Job title: {{job_title}}
Job description: {{job_description}}

Return ONLY valid JSON:
{
  "inferred_pain_point": "<one sentence>",
  "buying_signal_strength": "<low|medium|high>"
}
```

Feed `inferred_pain_point` into the 3b prompt as additional context so `recommended_angle` can
reference it directly (e.g. "They're hiring a Sales Ops Manager — likely scaling outbound;
mention how [product] cuts onboarding time for new reps").

---

## 3d. Deduplication

Run this check immediately before the `leads_enriched` insert, not earlier — you want dedup to
happen against the fully enriched record set, not raw scrape output (a lead might arrive from two
different sources with slightly different raw fields but resolve to the same company).

```sql
SELECT id FROM leads_enriched
WHERE website = $1 OR (company_name = $2 AND location = $3)
LIMIT 1;
```

**n8n pattern:**
```
[IF node: dedup match found]
   → true: [Supabase Update] set last_seen_at = now(), source_count = source_count + 1
   → false: [Supabase Insert] new row into leads_enriched
```

For contact-level dedup (same company, different named contact), key on `raw_email` OR
`raw_linkedin_url` rather than company alone — a company can legitimately have multiple valid
contacts.

---

## Scoring Rubric Guidance

A 1–10 ICP score is only useful if it's calibrated consistently. Before running a campaign at
scale:
1. Hand-score 20 known-good and 20 known-bad leads for the client's ICP.
2. Run the same 40 through the LLM scorer.
3. Compare — if the LLM's scores don't roughly rank-order the way the manual scores did, revise
   the ICP definition text (usually the fix is making disqualifiers more explicit) rather than
   the prompt structure.
4. Re-run until LLM/manual agreement is high enough to trust for the `min_icp_score` delivery
   threshold.

Re-calibrate whenever the client changes their ICP definition or after any noticeable drift in
lead quality feedback from the sales team.
