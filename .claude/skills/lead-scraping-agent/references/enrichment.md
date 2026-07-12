# Enrichment Layer — Prompt Templates, API Setup, Scoring Rubrics

Reference for Layer 3, where raw scraped rows become qualified, contextualized leads. This is
the only layer with an LLM in the hot path: Claude (`claude-sonnet-4-6`) runs the ICP fit
scoring in 3b, the intent inference in 3c, and — most importantly — generates the
`recommended_angle` outreach hook. Everything else in this layer is deterministic API plumbing.

**Storage note:** examples below use SQL/Supabase table names for precision. For a new client the
default backend is Google Sheets — `campaign_config` is the `Client Config` tab and
`leads_enriched` is the `Leads` tab in `storage-google-sheets.md`. The dedup query in 3d has a
Sheets-native equivalent described there.

---

## 3a. Company Enrichment API Setup

Firmographic enrichment runs *before* the LLM, deliberately: Claude can only score what it's
shown, and a scraped record alone is usually too thin to score honestly. Ground the record
first.

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
TTL. Firmographics move on a quarterly timescale; when five contacts share one company, paying
for five identical lookups is pure waste.

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

**Why this field needs watching.** Three of the four output fields are extractive — the score,
reasons, and disqualifiers should fall mechanically out of comparing the profile to the ICP
definition, and almost any competent model produces them. `recommended_angle` is different:
it's the single genuinely generative output in the whole pipeline, and the only field a
prospect ever indirectly sees (it becomes the icebreaker in outbound copy — delivery.md §5b).
The failure mode here is subtle — the JSON validates, the score is fine, and the angle is a
generic "I noticed your company is growing" that could apply to anyone. A good angle reasons
through *why* the company fits before writing the hook, so it cites the same specific evidence
that drove the score rather than defaulting to a generic opener — that's the thing to check for
in QA.

**Model call settings:**
- Use a low temperature (0–0.3) — the scoring fields need run-to-run consistency, and a low
  temperature still leaves Claude room to produce sufficiently specific angles.
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
criteria into the prompt string itself. The client will change their ICP; a config row update
should be all that takes.

---

## 3c. Intent Signal Detection

When the source is a job board, run a lighter Claude call over the posting text *before* the
main ICP scoring call:

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

Feed `inferred_pain_point` into the 3b prompt as additional context. This is how the
`recommended_angle` gets from "generic but true" to "specific and timely" — e.g. "They're
hiring a Sales Ops Manager — likely scaling outbound; mention how [product] cuts onboarding
time for new reps". A hiring signal is the company telling you its problem in public; the two
chained prompts just translate it into an opener.

---

## 3d. Deduplication

Run this check immediately before the `leads_enriched` insert, not earlier. The reason for the
late placement: the same company can arrive from two different sources with slightly different
raw fields, and only after enrichment do both records resolve to the same normalized company.
Dedup against raw scrape output misses these.

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
contacts, and collapsing them loses real leads.

---

## Scoring Rubric Guidance

A 1–10 score is only as good as its calibration — an uncalibrated scorer is worse than none,
because it launders guesses into numbers the client will trust. Before running a campaign at
scale:

1. Hand-score 20 known-good and 20 known-bad leads for the client's ICP.
2. Run the same 40 through the Claude scorer.
3. Compare — if the model's scores don't roughly rank-order the way the manual scores did,
   revise the ICP definition text rather than the prompt structure. In practice the fix is
   almost always making disqualifiers more explicit; models are generous scorers until told
   exactly what rules a company out.
4. Re-run until model/manual agreement is high enough to trust for the `min_icp_score` delivery
   threshold.

Re-calibrate whenever the client changes their ICP definition or after any noticeable drift in
lead quality feedback from the sales team. Calibration is not a one-time setup step — the sales
team's replies are the ground truth the scorer should keep converging toward.
