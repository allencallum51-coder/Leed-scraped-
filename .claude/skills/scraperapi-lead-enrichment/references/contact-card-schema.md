# Contact Card Schema

Field definitions and source hints for lead enrichment. Use when extracting data and deciding confidence levels.

## Person Fields

| Field | Source hints | Notes |
|-------|-------------|-------|
| **Full name** | Company team page, bio, press mention, LinkedIn, social profile | Prefer the most authoritative or official source |
| **Job title** | Company website team page, bio, LinkedIn headline, press releases | Current role only; skip past titles |
| **Company** | Email domain, company website, bio, any profile page | Current employer |
| **Email** | Company website /contact, Google search, press releases, email pattern inference | Work email preferred; personal only if work not available |
| **Phone** | Company website /contact, Google search, business directories | Work number preferred |
| **Profile URL** | Any public profile: LinkedIn, Twitter, GitHub, personal site, speaker page | Note the platform alongside the URL |
| **Location** | Company website, bio, press release, any profile page | City + country; use company HQ as fallback |
| **Twitter / X handle** | Google search, LinkedIn profile links, company website footer | Handle only (e.g., `@handle`), not full URL |
| **Other social** | Company footer links, Google search | GitHub for technical roles, YouTube for content creators, etc. |

## Company Fields

| Field | Source hints | Notes |
|-------|-------------|-------|
| **Company name** | Company website, Crunchbase, Wikipedia, any profile | Legal or trading name |
| **Domain** | Official website URL, email domain | Strip `www.`; e.g., `acme.com` |
| **LinkedIn company page** | Google search `site:linkedin.com/company` | Full URL; leave blank if not found |
| **Industry / vertical** | Crunchbase category, G2 category, company About page | Most specific label available |
| **Founded year** | Company About page, Crunchbase, Wikipedia, press releases | 4-digit year |
| **Headquarters** | Company website, Crunchbase, Wikipedia, About page | City + country |
| **Employee count** | LinkedIn company page, Crunchbase, Glassdoor, job listings | Range preferred (e.g., "51–200"); note source |
| **Revenue** | Crunchbase, press releases, Google search `"company" revenue`, SimilarWeb | Range or exact; note if estimated |
| **Total funding** | Crunchbase, AngelList/Wellfound, news articles | USD amount (e.g., "$12.4M") |
| **Latest funding round** | Crunchbase, news articles, press releases | Type (Seed, Series A, …) + amount + date |
| **Key investors** | Crunchbase, AngelList, press releases | Up to 5; lead investors first |
| **Technology stack** | Company job postings, GitHub org, company engineering blog, BuiltWith, footer badges | Key languages, frameworks, cloud platforms |
| **Key executives** | Company About/Team page, press releases, LinkedIn company (if available) | CEO and at least one technical/product leader |
| **Hiring signals** | Company careers page, Indeed, Glassdoor, LinkedIn jobs | "Actively hiring in [dept]" or "no recent postings found" |
| **Recent news** | Google News structured results, press release pages | Up to 3 headlines with dates; prefer funding, product, or leadership news |
| **Competitors** | Crunchbase "Similar companies", G2/Capterra alternatives, industry reports | Up to 5; note source |

## Confidence Levels

Append to any field value when presenting:

- **✓** Confirmed — found in 2+ independent sources, or returned by a structured endpoint
- **~** Likely — single source with strong contextual support (e.g., company About page says "Jane Doe, CEO")

Example: `Email: john.doe@acme.com ?`

When all high-priority fields (name, title, company, email or phone, location) are confirmed or likely, the enrichment is considered complete.
