# Delivery Layer — CRM Mappings, Outbound APIs, Dashboard Schema

Reference for Layer 5. Delivery is what the client actually touches — get the field mapping and
gating right, since mistakes here are the most visible to the client.

---

## 5a. CRM Field Mappings

**HubSpot (Create Contact + Create Company):**
| `leads_enriched` field | HubSpot property |
|---|---|
| `raw_contact_name` | `firstname` / `lastname` (split) |
| `raw_email` | `email` |
| `raw_contact_role` | `jobtitle` |
| `raw_linkedin_url` | custom property `linkedin_url` |
| `company_name` | Company object `name` |
| `website` | Company object `domain` |
| `icp_score` | custom property `icp_score` (number) |
| `recommended_angle` | custom property `recommended_outreach_angle` (single-line text) |
| `source` | custom property `lead_source` |

Use the HubSpot API node's **batch upsert** endpoint (`/crm/v3/objects/contacts/batch/upsert`)
keyed on email, not a per-record Create call — avoids duplicate contact records when a lead is
re-processed (e.g. `last_seen_at` update from dedup layer).

**Pipedrive (Create Person + Create Organization):**
| `leads_enriched` field | Pipedrive field |
|---|---|
| `raw_contact_name` | Person `name` |
| `raw_email` | Person `email` |
| `raw_contact_role` | custom field (create in Pipedrive settings first) |
| `company_name` | Organization `name`, linked via `org_id` on the Person |
| `icp_score` | custom field, number type |
| `recommended_angle` | custom field, large text type |

Create the Organization first, capture its `id`, then create the Person with `org_id` set —
Pipedrive doesn't auto-link on name match.

**Airtable (append to base):**
Simplest option — one row per lead in a base with columns matching `leads_enriched` directly.
Good for early-stage clients without a CRM yet, or as a lightweight review layer before a CRM
push (client approves a batch in Airtable, a separate n8n workflow syncs approved rows to the
real CRM).

---

## 5b. Outbound Sequence Injection

**Smartlead / Instantly:**
- Both expose a "add lead to campaign" API endpoint accepting the lead's email plus custom
  variables for personalization merge tags.
- Map `recommended_angle` to a custom variable (e.g. `{{icebreaker}}`) referenced in the email
  sequence templates — this is what makes the LLM enrichment step pay off in the outbound copy.

**Gating rule:**
```
[IF: icp_score >= campaign_config.min_icp_score]
   → true:  [HTTP Request: add to Smartlead/Instantly campaign]
   → false: [route to CRM only, no outbound injection]
```
Never inject unscored or low-score leads into an active sending campaign — a handful of bad
matches is what tanks sender reputation and deliverability for every future send.

---

## 5c. Dashboard

Minimal Next.js + Supabase dashboard reading directly from the same tables the pipeline writes
to (`leads_enriched`, `scraping_errors`, `source_health`).

**Pages / widgets:**
- **Lead volume over time** — daily count from `leads_enriched.created_at`, line chart.
- **Source breakdown** — count grouped by `source`, bar or donut chart.
- **ICP score distribution** — histogram of `icp_score` across current campaign.
- **Enrichment coverage %** — `count(enriched) / count(leads_raw)` — flags if the enrichment
  layer is silently falling behind extraction volume.
- **Self-healing alerts** — recent rows from `scraping_errors` where `resolved = false`, and the
  latest `source_health` pass/fail per source.

Use Supabase's client library directly from Next.js server components with row-level security
scoped to the client's `campaign_id` if the dashboard will ever serve more than one client from
the same Supabase project.

---

## 5d. Scheduled Export (Fallback)

Only use when CRM integration genuinely isn't possible (client has no CRM and doesn't want one
yet, or a procurement delay blocks API access).

```
[Schedule Trigger: weekly]
        ↓
[Supabase: select leads_enriched where icp_score >= min_icp_score and delivered = false]
        ↓
[Spreadsheet File node: convert to CSV]
        ↓
[Send Email node: attach CSV]
        ↓
[Supabase Update: mark delivered = true]
```

Communicate clearly to the client that this is a stopgap — the pitch for upgrading to CRM
delivery (5a) is that a CSV in an inbox never gets touched by the sales team the way a native CRM
record does.
