# Leed-scraped- (Lead Workshop)

This repo is the build system for bespoke, per-client lead-generation/scraping agents. It is
not a single client's scraper — it's the reusable factory that produces one, configured per
client.

**Start here:** the `lead-scraper-creator` skill (`.claude/skills/lead-scraper-creator/SKILL.md`)
is the front door. Mention "the lead scraper creator," "the workshop," or ask to build/spec a
lead-gen agent for a client, and it loads the full system. It indexes:

- `lead-scraping-agent` — the five-layer build guide (Source → Extraction → Enrichment/Scoring →
  Self-Healing → Delivery) and its six reference docs.
- `prototypes/intake-wizard.html` — the client-intake wizard prototype.
- ~18 complementary installed skills (gmaps-leads, apollo-lead-finder, icp-identification,
  scraperapi-*, etc.), grouped by which pipeline layer they support.

## Standing decisions (don't relitigate these without being asked)

- **Storage default is Google Sheets**, one workbook per client — not Supabase. Supabase is the
  documented scale path once a client outgrows Sheets (see
  `references/storage-google-sheets.md` for the migration criteria). Column names are identical
  across both so migration is copy-rows, not redesign.
- **The pipeline's enrichment/scoring LLM is Claude** (`claude-sonnet-4-6`). This system's own
  documentation was rewritten using Fable 5 as a one-time authoring exercise — that never
  changed, and shouldn't change again, what model the actual pipeline calls at runtime.
- Real scraping/enrichment API credentials (Google Places, Apify, ScraperAPI, Clearbit, Hunter,
  etc.) are not configured in this environment. Anything presented as "scraped" without those
  keys is manual web research standing in for the real pipeline — always say so explicitly.

## Branches

There is currently only one branch, `claude/vercel-labs-skills-cef34c`, and it is the repo's
default (HEAD) branch. Any new session that checks out this repo gets it automatically — no PR
or merge is needed for that to be true.

## Where memory actually lives

Nothing in a chat's context window is durable. This file, the skill files, and the reference
docs are: they're plain files in this git repo, so they survive every context reset, compaction,
and new session — as long as the new session has this same repo checked out. When you start a
fresh chat and want continuity, point it at this repo and reference `lead-scraper-creator` (or
just ask a question that matches one of the skill descriptions) rather than trying to re-explain
prior context from memory. If a decision should survive future resets, it belongs in one of
these files, not just in a conversation.
