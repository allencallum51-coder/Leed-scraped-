# ScraperAPI Node — Field Reference

Authoritative list of every parameter the official ScraperAPI n8n node
accepts, as the names appear in workflow JSON. Source: the public
`n8n-nodes-scraperapi-official` package on npm.

Three top-level `resource` values: `api`, `sde`, `crawler`. Pick one,
then pick `operation` (and for `sde`, also `sdePlatform`), then fill
required fields plus the operation-specific optional collection.

## Resource: `api`

```json
{
  "resource": "api",
  "operation": "apiRequest",
  "apiUrl": "https://example.com",
  "apiOptionalParameters": {
    "apiAutoparse": false,
    "apiCountryCode": "us",
    "apiDesktopDevice": false,
    "apiMobileDevice": false,
    "apiOutputFormat": "html",
    "apiPremium": false,
    "apiRender": false,
    "apiUltraPremium": false
  }
}
```

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `apiUrl` | yes | string | Target URL |
| `apiAutoparse` | no | boolean | Auto-parse supported sites into JSON. Returns JSON instead of HTML. |
| `apiCountryCode` | no | string | Two-letter ISO code (e.g., `us`, `gb`, `de`) |
| `apiDesktopDevice` | no | boolean | Mutually exclusive with `apiMobileDevice` |
| `apiMobileDevice` | no | boolean | Mutually exclusive with `apiDesktopDevice` |
| `apiOutputFormat` | no | enum | `html` (default), `markdown`, `text`, `csv`, `json`. `csv` and `json` require an autoparse-supported site. |
| `apiPremium` | no | boolean | Mutually exclusive with `apiUltraPremium` |
| `apiRender` | no | boolean | JS rendering. +10 credits. |
| `apiUltraPremium` | no | boolean | Highest-tier proxies. +30 credits. |

## Resource: `sde` (Structured Data Endpoint)

Choose a `sdePlatform`, then choose an `operation` valid for that
platform, then fill the operation's required field plus the per-operation
optional collection.

```json
{
  "resource": "sde",
  "sdePlatform": "amazon",
  "operation": "amazonProduct",
  "sdeAsin": "B08N5WRWNW",
  "sdeAmazonProductOptions": {
    "tld": "com",
    "countryCode": "us",
    "outputFormat": "json"
  }
}
```

### Required field per operation

| `sdePlatform` | `operation` | Required field |
|---------------|-------------|----------------|
| `amazon` | `amazonProduct` | `sdeAsin` |
| `amazon` | `amazonOffers` | `sdeAsin` |
| `amazon` | `amazonSearch` | `sdeQuery` |
| `google` | `googleSearch` | `sdeQuery` |
| `google` | `googleJobs` | `sdeQuery` |
| `google` | `googleNews` | `sdeQuery` |
| `google` | `googleShopping` | `sdeQuery` |
| `google` | `googleMapsSearch` | `sdeQuery` |
| `ebay` | `ebaySearch` | `sdeQuery` |
| `ebay` | `ebayProduct` | `sdeProductId` |
| `walmart` | `walmartSearch` | `sdeQuery` |
| `walmart` | `walmartCategory` | `sdeCategory` |
| `walmart` | `walmartProduct` | `sdeProductId` |
| `walmart` | `walmartReview` | `sdeProductId` |
| `redfin` | `redfinForSale` | `sdeUrl` |
| `redfin` | `redfinForRent` | `sdeUrl` |
| `redfin` | `redfinSearch` | `sdeUrl` |
| `redfin` | `redfinAgent` | `sdeUrl` |

### Optional-parameters collection per operation

Each operation has a uniquely-named collection. Fields inside the
collection (e.g., `tld`, `countryCode`, `outputFormat`) are flat —
not prefixed.

| `operation` | Collection name | Common fields |
|-------------|-----------------|---------------|
| `amazonProduct` | `sdeAmazonProductOptions` | `tld`, `countryCode`, `outputFormat` |
| `amazonOffers` | `sdeAmazonOffersOptions` | `tld`, `countryCode`, `outputFormat`, `condition`, `filterNew`, `filterUsedAcceptable`, `filterUsedGood`, `filterUsedLikeNew`, `filterUsedVeryGood` |
| `amazonSearch` | `sdeAmazonSearchOptions` | `tld`, `countryCode`, `outputFormat`, `department`, `page`, `sort` |
| `googleSearch` | `sdeGoogleSearchOptions` | `tld`, `countryCode`, `outputFormat`, `includeHtml`, `timePeriod` |
| `googleShopping` | `sdeGoogleShoppingOptions` | `tld`, `countryCode`, `outputFormat`, `includeHtml` |
| `googleNews` | `sdeGoogleNewsOptions` | `tld`, `countryCode`, `outputFormat`, `timePeriod` |
| `googleJobs` | `sdeGoogleJobsOptions` | `tld`, `countryCode`, `outputFormat` |
| `googleMapsSearch` | `sdeGoogleMapsSearchOptions` | `tld`, `countryCode`, `includeHtml`, `latitude`, `longitude`, `zoom` |
| `ebayProduct` | `sdeEbayProductOptions` | `tld`, `countryCode`, `outputFormat` |
| `ebaySearch` | `sdeEbaySearchOptions` | `tld`, `countryCode`, `outputFormat`, `buyingFormat`, `condition`, `itemsPerPage`, `page`, `sellerId`, `showOnly`, `sortBy` |
| `walmartProduct` | `sdeWalmartProductOptions` | `tld`, `countryCode`, `outputFormat` |
| `walmartSearch` / `walmartCategory` | `sdeWalmartSearchOptions` | `tld`, `countryCode`, `outputFormat`, `page` |
| `walmartReview` | `sdeWalmartReviewOptions` | `tld`, `countryCode`, `outputFormat`, `page`, `ratings`, `sort`, `verifiedPurchase` |
| `redfinForSale` / `redfinForRent` | `sdeRedfinListingOptions` | `tld`, `countryCode`, `raw` |
| `redfinSearch` / `redfinAgent` | `sdeRedfinLookupOptions` | `tld`, `countryCode` |

### `tld` allowed values per platform

- **Amazon:** `com`, `co.uk`, `ca`, `de`, `es`, `fr`, `ie`, `it`, `co.jp`,
  `co.za`, `in`, `cn`, `com.sg`, `com.mx`, `ae`, `com.br`, `nl`, `com.au`,
  `com.tr`, `sa`, `se`, `pl`
- **Google:** same as Amazon minus `ie` and `co.za`
- **eBay:** `com`, `co.uk`, `com.au`, `de`, `ca`, `fr`, `it`, `es`, `at`,
  `ch`, `com.sg`, `com.my`, `ph`, `ie`, `pl`, `nl`
- **Walmart:** `com`, `ca`
- **Redfin:** `com`, `ca`

### Enum values worth noting

- `apiOutputFormat` / `sde* outputFormat`: `csv`, `html`, `json`,
  `markdown`, `text` (SDE collections only expose `csv` and `json`)
- `timePeriod`: `qdr:h` (past hour), `qdr:d` (past day), `qdr:w` (past
  week), `qdr:m` (past month), `qdr:y` (past year)
- `ebaySearch.buyingFormat`: `accepts_offers`, `auction`, `buy_it_now`
- `ebaySearch.condition`: `for_parts`, `new`, `not_working`, `open_box`,
  `refurbished`, `used`
- `ebaySearch.sortBy`: `best_match`, `distance_nearest`, `ending_soonest`,
  `newly_listed`, `price_highest`, `price_lowest`
- `walmartReview.sort`: `helpful`, `submission-desc`, `submission-asc`,
  `rating-desc`, `rating-asc`, `relevancy`

## Resource: `crawler`

Three operations. `crawlerJobCreate` does the work; the other two are
admin.

### `crawlerJobCreate`

```json
{
  "resource": "crawler",
  "operation": "crawlerJobCreate",
  "crawlerStartUrl": "https://example.com",
  "crawlerMaxDepth": 2,
  "crawlerUrlRegexpInclude": ".*",
  "crawlerCallbackUrl": "https://your-n8n.example.com/webhook/scrapera",
  "crawlerOptionalParameters": {
    "crawlerApiParameters": {
      "crawlerApiAutoparse": false,
      "crawlerApiCountryCode": "us",
      "crawlerApiDesktopDevice": false,
      "crawlerApiMobileDevice": false,
      "crawlerApiOutputFormat": "html",
      "crawlerApiPremium": false,
      "crawlerApiRender": false,
      "crawlerApiUltraPremium": false
    },
    "crawlerEnabled": true,
    "crawlerScheduleInterval": "once",
    "crawlerScheduleName": "my-crawler",
    "crawlerUrlRegexpExclude": ""
  }
}
```

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `crawlerStartUrl` | yes | string | Depth-0 URL |
| `crawlerMaxDepth` | one of | number | Either this or `crawlerCrawlBudget` must be set |
| `crawlerCrawlBudget` | one of | number | Cap on credits this job may consume |
| `crawlerUrlRegexpInclude` | yes | string | Defaults to `.*`. Crawl all links matching this regex. |
| `crawlerCallbackUrl` | yes | string | Webhook URL where results are POSTed. Must be reachable from ScraperAPI. |
| `crawlerOptionalParameters.crawlerApiParameters.*` | no | object | Per-page scrape settings (same fields as the `api` resource but with `crawlerApi*` prefix) |
| `crawlerOptionalParameters.crawlerEnabled` | no | boolean | `false` creates the config but doesn't run it. Default `true`. |
| `crawlerOptionalParameters.crawlerScheduleInterval` | no | enum | `once`, `hourly`, `daily`, `weekly`, `monthly` |
| `crawlerOptionalParameters.crawlerScheduleName` | no | string | Identifies the schedule in the ScraperAPI dashboard |
| `crawlerOptionalParameters.crawlerUrlRegexpExclude` | no | string | Skip URLs matching this regex |

### `crawlerJobGet`

```json
{ "resource": "crawler", "operation": "crawlerJobGet", "crawlerJobId": "..." }
```

### `crawlerJobDelete`

```json
{ "resource": "crawler", "operation": "crawlerJobDelete", "crawlerJobId": "..." }
```

## Credential block

The credential's internal key in workflow JSON is `scraperApi-Api`. The
human-readable name shown in the n8n UI is "ScraperAPI API". Always
include this block on every ScraperAPI node:

```json
"credentials": {
  "scraperApi-Api": {
    "id": "REPLACE_WITH_CREDENTIAL_ID",
    "name": "ScraperAPI API"
  }
}
```

The `id` is instance-local — never copy a real ID from one n8n instance
into JSON intended for another. Imported workflows will surface a "select
credential" warning that the user resolves once by picking their existing
credential from the dropdown.

## Node output shape

Every operation returns one item per input item:

```json
{
  "resource": "api",
  "response": {
    "body": "...",
    "headers": { },
    "statusCode": 200,
    "statusMessage": "OK"
  }
}
```

- `api` and `crawler`: `body` is the raw payload string
- `sde`: `body` is the parsed JSON document, **as a string** — parse it
  in a Code node before downstream nodes can read individual fields

## Documentation

- Public node listing: `https://n8n.io/integrations/scraperapi/`
- npm package: `https://www.npmjs.com/package/n8n-nodes-scraperapi-official`
- ScraperAPI API docs: `https://docs.scraperapi.com/`
- Crawler API docs: `https://docs.scraperapi.com/scraperapi-crawler-v2.0/crawler-api/job-lifecycle`
