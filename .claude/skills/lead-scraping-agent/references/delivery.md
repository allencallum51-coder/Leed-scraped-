# Delivery Layer — CRM Mappings, Outbound APIs, Dashboard Schema

Reference for Layer 5. Delivery is the only layer the client touches, which changes the error
economics: a bug in extraction costs you a re-run, a bug here costs you credibility. Get the
field mappings and the gating exactly right before anything goes live.

**Storage note:** until a client's CRM/mail-service integration is actually wired up, `delivery_target`
and `outbound_injection_threshold` are just fields saved on the `Client Config` tab of the
client's Google Sheet (see `storage-google-sheets.md`) — no live connection exists yet. The
gating logic below (5b) still applies once outbound injection goes live; `contact_status` and
`opted_out` on the `Leads` tab are what it reads.

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
keyed on email, not a per-record Create call. The dedup layer legitimately re-processes leads
(the `last_seen_at` update path), and per-record Creates turn every re-process into a duplicate
contact in the client's CRM.

**Pipedrive (Create Person + Create Organization):**
| `leads_enriched` field | Pipedrive field |
|---|---|
| `raw_contact_name` | Person `name` |
| `raw_email` | Person `email` |
| `raw_contact_role` | custom field (create in Pipedrive settings first) |
| `company_name` | Organization `name`, linked via `org_id` on the Person |
| `icp_score` | custom field, number type |
| `recommended_angle` | custom field, large text type |

Order matters here: create the Organization first, capture its `id`, then create the Person
with `org_id` set — Pipedrive doesn't auto-link on name match, and unlinked Persons are how a
CRM quietly rots.

**Airtable (append to base):**
Simplest option — one row per lead in a base with columns matching `leads_enriched` directly.
Two situations where it's the right call: early-stage clients who don't have a CRM yet, and as
a human-review buffer in front of a real CRM (client approves a batch in Airtable, a separate
n8n workflow syncs approved rows onward).

---

## 5b. Outbound Sequence Injection

**Smartlead / Instantly:**
- Both expose a "add lead to campaign" API endpoint accepting the lead's email plus custom
  variables for personalization merge tags.
- Map `recommended_angle` to a custom variable (e.g. `{{icebreaker}}`) referenced in the email
  sequence templates. This is the moment the whole Layer 3 investment pays off — the Claude
  generated hook lands in the first line of the actual email a prospect reads.

**Gating rule:**
```
[IF: icp_score >= campaign_config.min_icp_score
     AND opted_out = FALSE
     AND email NOT IN Suppression List]
   → true:  [HTTP Request: add to Smartlead/Instantly campaign]
            → set contact_status = 'messaged', last_contacted_at = now(),
              message_count += 1
   → false: [route to CRM only, no outbound injection]
```

Three checks, two different reasons:

- The **score gate** protects sender reputation. A handful of obviously-bad matches in an
  active sequence is what earns spam reports, and spam reports tank deliverability for every
  future send. Unscored or low-score leads go to the CRM, never into a live campaign.
- The **opt-out and suppression checks** are compliance, not optimization — they are
  non-negotiable regardless of score. A 10/10 ICP fit does not override a prior removal
  request.

**Handling an opt-out reply:** whatever parses inbound replies (Smartlead/Instantly webhook, or
a manual inbox check) must, the moment it detects an unsubscribe/removal request, do both of:
set `Leads.opted_out = TRUE` + `opted_out_at = now()` on that row, **and** append the email to
the `Suppression List` tab. They cover different failure modes — the row flag stops this
campaign from re-sending; the suppression entry stops any future campaign from re-adding the
same person after the row is gone.

---

## 5c. Dashboard

Minimal Next.js + Supabase dashboard reading directly from the same tables the pipeline writes
to (`leads_enriched`, `scraping_errors`, `source_health`) — no separate analytics store to keep
in sync.

**Pages / widgets:**
- **Lead volume over time** — daily count from `leads_enriched.created_at`, line chart.
- **Source breakdown** — count grouped by `source`, bar or donut chart.
- **ICP score distribution** — histogram of `icp_score` across current campaign.
- **Enrichment coverage %** — `count(enriched) / count(leads_raw)` — the early-warning metric;
  a falling ratio means the enrichment layer is silently losing ground to extraction volume.
- **Self-healing alerts** — recent rows from `scraping_errors` where `resolved = false`, and the
  latest `source_health` pass/fail per source.

Use Supabase's client library directly from Next.js server components with row-level security
scoped to the client's `campaign_id` if the dashboard will ever serve more than one client from
the same Supabase project. Retrofit RLS later and you'll be auditing every query instead of
writing one policy.

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

Frame it to the client as a stopgap from day one, and keep the CRM upgrade (5a) on the table.
The honest pitch: a CSV in an inbox gets opened once and forgotten; a native CRM record sits in
the sales team's working view every day. Same leads, very different usage — and usage is what
gets the engagement renewed.
