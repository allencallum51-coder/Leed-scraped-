# Storage — Google Sheets (Default Backend)

Reference for the default backing store used by a new client's build. Every other reference
file (`extraction.md`, `enrichment.md`, `self-healing.md`, `delivery.md`) describes tables in
Postgres/Supabase terms because that's the schema a client migrates to at scale — this file is
the Google Sheets equivalent to use from day one. Column names match 1:1 across both so migrating
a client later is copying rows into the equivalent Postgres columns, not redesigning anything.

**One workbook per client**, named `{client_name} — Lead Pipeline`, with five tabs.

---

## Tab 1 — `Leads`

The merged equivalent of `leads_raw` + `leads_enriched`. Sheets has no benefit from splitting
raw/enriched into separate physical tables the way Postgres does for write-locking reasons — a
single `enrichment_status` column does that job instead.

| Column | Notes |
|---|---|
| `lead_id` | Stable ID — hash of `website` + `company_name`, generated on first insert. Used for dedup and as the row's stable key across re-scrapes. |
| `company_name` | |
| `website` | |
| `industry` | |
| `location` | |
| `employee_count_range` | |
| `source` | Which configured source produced this row. |
| `scraped_at` | ISO timestamp of first scrape. |
| `last_seen_at` | Updated instead of creating a duplicate row when a later scrape re-finds the same lead (see Dedup below). |
| `raw_contact_name` | |
| `raw_contact_role` | |
| `raw_email` | |
| `raw_linkedin_url` | |
| `email_validation_status` | `valid` / `invalid` / `catch-all` / `unknown` — from ZeroBounce/NeverBounce. |
| `enrichment_status` | `raw` / `enriched` / `schema_error` — gates whether this row is eligible for delivery. |
| `icp_score` | 1–10, from the Layer 3 scoring prompt. |
| `fit_reasons` | Semicolon-joined list (Sheets cells don't nest arrays). |
| `disqualifiers` | Semicolon-joined list. |
| `recommended_angle` | One-sentence personalized hook. |
| **`contact_status`** | `not_contacted` / `messaged` / `replied` / `bounced` / `meeting_booked` / `disqualified` — the outreach-tracking column. Updated by whatever sends the actual message (Smartlead/Instantly webhook, or manual update) — the scraping pipeline only ever sets it to `not_contacted` on insert. |
| **`last_contacted_at`** | ISO timestamp of the most recent message sent to this lead. |
| **`message_count`** | Increments each time an outbound touch goes out — lets you cap follow-ups per lead without a separate table. |
| **`opted_out`** | `TRUE` / `FALSE`. Set the moment a reply contains an unsubscribe/removal request. This is the field every outbound step must check before sending — see Suppression below. |
| `opted_out_at` | ISO timestamp, blank until `opted_out` flips to `TRUE`. |
| `delivered_to_crm` | `TRUE` / `FALSE` — placeholder column; stays `FALSE` for every client until a real CRM/mail-service integration is wired up (see `delivery.md`). Not the same thing as `contact_status` — a lead can be delivered to a CRM without having been messaged yet. |

---

## Tab 2 — `Suppression List`

A permanent record of every email/domain that has ever opted out, independent of the `Leads` tab.
The reason this is a separate tab rather than relying on `opted_out` alone: if a lead is later
removed from `Leads` (e.g. a sheet cleanup) and the same person is re-scraped in a future
campaign, `Leads.opted_out` alone won't catch it — the Suppression tab is the permanent
cross-campaign check.

| Column | Notes |
|---|---|
| `email` | Lowercased, trimmed — the match key. |
| `domain` | Optional broader suppression (e.g. suppress an entire company). |
| `opted_out_at` | |
| `reason` | `unsubscribe_request` / `bounced_hard` / `manual` |
| `source_campaign` | Which client/campaign this opt-out came from, for audit purposes. |

**Every outbound-injection step (delivery.md §5b) must check both `Leads.opted_out = FALSE` and
absence from `Suppression List` before sending** — the Sheets version of a hard gate.

---

## Tab 3 — `Client Config`

One row per client, flattened from the intake wizard's JSON config (nested arrays become
comma-joined cells):

| Column | Source (wizard field) |
|---|---|
| `client_name`, `industry`, `contact_name`, `contact_email`, `engagement_type`, `notes` | Step 1 |
| `icp_definition`, `target_roles`, `company_size_min`, `company_size_max`, `geography`, `disqualifiers`, `min_icp_score` | Step 2 |
| `primary_source`, `fallback_source`, `enabled_sources` | Step 3 |
| `company_enrichment_provider`, `email_validation_provider`, `intent_signal_detection` | Step 4 |
| `delivery_target`, `delivery_status`, `outbound_injection_threshold` | Step 5 |
| `storage_backend` | `google_sheets` or `supabase` — set once and read by every n8n workflow so the same workflow definition works for either backend without a code change. |
| `generated_at` | Step 6 |

---

## Tab 4 — `Scraping Errors`

Sheets equivalent of the `scraping_errors` table in self-healing.md §4a.

| Column | Notes |
|---|---|
| `error_id`, `source`, `error_type`, `error_message`, `payload` (JSON string), `retry_count`, `resolved`, `created_at` | Same fields, same meaning as the SQL version. |

---

## Tab 5 — `Source Health`

Sheets equivalent of self-healing.md §4f.

| Column | Notes |
|---|---|
| `source`, `passed`, `details`, `checked_at` | |

---

## Working With Sheets Instead of SQL

**Dedup (enrichment.md §3d):** there's no `WHERE website = $1 OR (...)`. Read the full `Leads`
tab into memory at the start of a run (via `values.get` or a Sheets node's "read all rows"),
build a lookup keyed on `website` and on `company_name + location`, then check new rows against
that lookup before appending. At a few thousand rows this is fast; re-evaluate once a single
client's `Leads` tab approaches the tens of thousands.

**Batching:** the Sheets API is quota-limited (roughly 300 read/write requests per minute per
project, 60 per user per minute). Use batch operations (`spreadsheets.values.batchGet` /
`batchUpdate`, or n8n's Google Sheets node in "append/update many" mode) instead of one API call
per lead.

**Concurrency:** Sheets has no row-level locking. If more than one workflow could write to the
same `Leads` tab at the same time (e.g. a scheduled scrape and a manual re-run overlapping),
serialize writes through a single n8n workflow rather than relying on read-check-write per row —
two workflows racing on the same dedup check can both pass the check and insert a duplicate.

**No JSON columns:** anywhere the Postgres schema uses `JSONB` (e.g. `scraping_errors.payload`),
store a JSON *string* in the Sheets cell and parse it in the workflow when needed, rather than
trying to flatten it into more columns.

---

## Migration Path to Supabase

Move a client from Sheets to Supabase when any of the following starts being true:
- `Leads` tab is approaching ~10,000+ active rows and dedup/read-all-rows is visibly slowing the
  pipeline down.
- More than one workflow needs to write concurrently and the serialization workaround above is
  becoming a bottleneck.
- The client wants the dashboard (delivery.md §5c) with real-time queries/joins across leads,
  errors, and health — Sheets can back a simple dashboard but not one doing relational joins at
  speed.

Because every column name in this file matches its Postgres counterpart in the other reference
docs, migration is: create the Supabase tables from the existing schema definitions, then copy
each Sheets tab's rows into the matching table. No field renaming, no logic changes in the n8n
workflows beyond swapping the Sheets nodes for Supabase nodes — `storage_backend` in
`Client Config` is what a workflow reads to know which node path to take.
