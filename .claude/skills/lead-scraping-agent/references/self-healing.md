# Self-Healing Layer — n8n Blueprints

Reference for Layer 4. Each subsection below maps directly to the strategies listed in the main
SKILL.md (4a–4f).

---

## 4a. Extraction Failure Detection

Wrap every extraction-layer HTTP Request / Apify actor call node with an error output branch
(n8n: enable "Continue On Fail" on the node, then branch on `$json.error`).

```
[HTTP Request (Continue On Fail: true)]
        ↓
[IF: $json.error exists]
   → true:  [Function: classify error type] → [Supabase Insert: scraping_errors]
                                             → [Execute Workflow: self-heal-subworkflow]
   → false: [continue extraction pipeline]
```

**Error classification function:**
```js
const status = $json.error?.httpCode;
let errorType;
if (status === 429) errorType = 'rate_limit';
else if (status === 401 || status === 403) errorType = 'auth_block';
else if (status >= 500 || $json.error?.code === 'ETIMEDOUT') errorType = 'timeout';
else errorType = 'dom_change';
return [{ json: { ...($json), error_type: errorType } }];
```

Insert into `scraping_errors` with the full request payload attached — you need it to replay the
exact failed request once the self-heal branch decides how to remediate.

---

## 4b. Retry With Backoff

Sub-workflow triggered from 4a for `rate_limit` and `timeout` error types:

```
[Execute Workflow Trigger]
        ↓
[Supabase: read retry_count for this error row]
        ↓
[Switch on retry_count]
   0 → [Wait: 5 minutes]  → [Retry original request] → success? resolve : increment + loop
   1 → [Wait: 30 minutes] → [Retry original request] → success? resolve : increment + loop
   2 → [Wait: 2 hours]    → [Retry original request] → success? resolve : mark unresolved
   3+ → [Slack/Email webhook: alert] → [Supabase: resolved = false]
```

Use n8n's **Wait node** (not a `sleep` in Function code — it suspends the workflow execution
rather than blocking a worker) so retries don't tie up execution slots for hours.

---

## 4c. DOM Change Detection & Auto-Remediation

**Detection (runs after every scrape of an HTML source):**
```
[HTTP Request: fetch target page]
        ↓
[Function: hash the critical selector's extracted output]
        ↓
[Supabase: compare against stored hash for this source]
        ↓
[IF: hash changed]
   → true: [Supabase Update: scraper_config.status = 'dom_change']
           → [pause this source: Set active = false]
           → [trigger LLM remediation sub-workflow]
```

**LLM remediation prompt:**
```
The following CSS selector no longer returns expected data from {{url}}.
Previous working selector: {{old_selector}}
Here is the current page HTML (truncated): {{html_snippet}}
Suggest the updated CSS selector that targets the equivalent element.
Return ONLY the new selector string.
```

**Confidence gate:** scan the LLM response for uncertainty language ("might", "possibly",
"I'm not sure", "unable to determine") before trusting it automatically.
```js
const uncertain = /\b(might|possibly|not sure|unable to|cannot determine|unclear)\b/i.test(
  $json.suggested_selector
);
return [{ json: { ...$json, requires_human_review: uncertain } }];
```
- `requires_human_review: false` → write the new selector to `scraper_config`, set
  `active = true`, resume the source on the next scheduled run.
- `requires_human_review: true` → leave the source paused, alert via Slack with the suggested
  selector attached for a human to approve or correct.

---

## 4d. Proxy Rotation

- Configure rotation at the proxy provider level (Bright Data / Oxylabs sticky-session pools or
  rotating pools, depending on whether the target needs session continuity).
- Store proxy credentials as n8n **credentials**, never hardcoded in HTTP Request nodes or
  Function code — this also means rotating the credential doesn't require touching every
  workflow that uses it.
- Maintenance workflow (separate, scheduled monthly or per provider's rotation policy):
  ```
  [Schedule Trigger] → [HTTP Request: provider's key-rotation endpoint]
                      → [n8n API: update credential] → [Slack: confirm rotation]
  ```

---

## 4e. Schema Drift Detection

Validate every record against a schema immediately before the `leads_enriched` write — this is
the last gate before data becomes "real" in the pipeline.

```js
// Function node, using Zod-equivalent manual checks (or a Zod-in-n8n Code node if available)
const required = ['company_name', 'icp_score', 'source'];
const missing = required.filter(k => $json[k] === undefined || $json[k] === null);
const typeErrors = [];
if (typeof $json.icp_score !== 'number') typeErrors.push('icp_score not a number');

if (missing.length || typeErrors.length) {
  return [{
    json: {
      ...$json,
      _validation_failed: true,
      _errors: [...missing.map(m => `missing: ${m}`), ...typeErrors]
    }
  }];
}
return [{ json: $json }];
```
```
[IF: _validation_failed]
   → true:  [Supabase Insert: schema_errors] → halt this record (do not write partial data)
   → false: [continue to leads_enriched insert]
```

---

## 4f. Health Check Workflow

Runs daily, independent of the main pipeline schedule:

```
[Schedule Trigger: daily 06:00]
        ↓
[Supabase: read active sources from scraper_config]
        ↓
[Loop over sources] → [run a single test scrape against a known-good target for that source]
        ↓
[Function: validate output matches expected schema shape]
        ↓
[Supabase Insert: source_health (source, passed, checked_at, details)]
        ↓
[Aggregate all results] → [Slack/Email: daily health summary]
```

**`source_health` schema:**
```sql
CREATE TABLE source_health (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  source TEXT NOT NULL,
  passed BOOLEAN NOT NULL,
  details JSONB,
  checked_at TIMESTAMPTZ DEFAULT now()
);
```

The daily summary is also the artifact that justifies a self-healing retainer to the client —
surface it (or a rollup of it) on the dashboard described in delivery.md.
