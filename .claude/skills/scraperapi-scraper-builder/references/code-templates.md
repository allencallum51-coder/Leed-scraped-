# Code Templates

Base templates for the `scraperapi-scraper-builder` skill. Copy and adapt — do not use as-is. The TODO comments mark where scraper-specific logic (URL construction, HTML parsing, data extraction) must be filled in.

---

## Python — Sync (< 100 URLs)

```python
# Requirements: pip install requests beautifulsoup4
# Usage: SCRAPERAPI_API_KEY=your_key python scraper.py --url "https://example.com/products" --pages 10

import os
import sys
import time
import json
import argparse
import requests
from typing import Optional

API_KEY = os.environ.get("SCRAPERAPI_API_KEY")
if not API_KEY:
    sys.exit("Error: SCRAPERAPI_API_KEY environment variable not set")

BASE_URL = "https://api.scraperapi.com/"


def scrape(url: str, params: dict, attempt: int = 0) -> Optional[str]:
    req_params = {"api_key": API_KEY, "url": url, **params}
    try:
        r = requests.get(BASE_URL, params=req_params, timeout=60)
        if r.status_code == 200:
            return r.text
        if r.status_code == 401:
            sys.exit("Error: Invalid API key (401)")
        if r.status_code == 403:
            raise RuntimeError("Blocked (403). Retry with --premium or --ultra-premium.")
        if r.status_code == 404:
            print(f"Not found (404): {url}", file=sys.stderr)
            return None
        if r.status_code in (429, 500, 503):
            if attempt < 4:
                wait = 2 ** attempt
                print(f"Retrying in {wait}s (status {r.status_code})...", file=sys.stderr)
                time.sleep(wait)
                return scrape(url, params, attempt + 1)
            raise RuntimeError(f"Max retries exceeded (status {r.status_code})")
        raise RuntimeError(f"Unexpected status: {r.status_code}")
    except requests.RequestException as e:
        if attempt < 4:
            time.sleep(2 ** attempt)
            return scrape(url, params, attempt + 1)
        raise


def estimate_credits(pages: int, render: bool, premium: bool, ultra_premium: bool) -> int:
    cost = 1
    if render:
        cost = 10
    if premium:
        cost = max(cost, 10)
    if ultra_premium:
        cost = 30
    return pages * cost


def build_page_url(base_url: str, page: int) -> str:
    # TODO: Adapt for the target site's pagination scheme
    # Common patterns:
    #   ?page=N          → f"{base_url}?page={page}"
    #   ?start=N*10      → f"{base_url}?start={(page-1)*10}"
    #   /page/N/         → f"{base_url.rstrip('/')}/page/{page}/"
    if page == 1:
        return base_url
    return f"{base_url}?page={page}"


def extract(html: str, url: str) -> list[dict]:
    # TODO: Replace with your actual parsing logic
    # from bs4 import BeautifulSoup
    # soup = BeautifulSoup(html, "html.parser")
    # return [
    #     {"title": el.get_text(strip=True), "url": el.find("a")["href"]}
    #     for el in soup.select("li.product-item")
    # ]
    return [{"url": url, "html_length": len(html)}]


def main():
    parser = argparse.ArgumentParser(description="Scraper powered by ScraperAPI")
    parser.add_argument("--url", required=True, help="Target URL")
    parser.add_argument("--pages", type=int, default=1, help="Pages to scrape")
    parser.add_argument("--output", help="Output file (default: stdout)")
    parser.add_argument("--max-credits", type=int, help="Abort if estimate exceeds this")
    parser.add_argument("--render", action="store_true", help="Enable JS rendering (~10 credits/req)")
    parser.add_argument("--premium", action="store_true", help="Use premium proxies (~10 credits/req)")
    parser.add_argument("--ultra-premium", action="store_true", help="Use ultra premium proxies (~30 credits/req)")
    parser.add_argument("--country-code", help="Proxy country (e.g. us, gb, de)")
    parser.add_argument("--session", type=int, help="Session number for IP stickiness")
    args = parser.parse_args()

    estimated = estimate_credits(args.pages, args.render, args.premium, args.ultra_premium)
    cost_per = estimated // args.pages
    print(f"Estimated: {args.pages} pages × {cost_per} credits = ~{estimated} credits", file=sys.stderr)

    if args.max_credits and estimated > args.max_credits:
        sys.exit(f"Error: Estimated {estimated} credits exceeds --max-credits={args.max_credits}")

    scrape_params: dict = {}
    if args.render:
        scrape_params["render"] = "true"
    if args.premium:
        scrape_params["premium"] = "true"
    if args.ultra_premium:
        scrape_params["ultra_premium"] = "true"
    if args.country_code:
        scrape_params["country_code"] = args.country_code
    if args.session:
        scrape_params["session_number"] = str(args.session)

    results = []
    for page in range(1, args.pages + 1):
        url = build_page_url(args.url, page)
        print(f"Scraping page {page}/{args.pages}: {url}", file=sys.stderr)
        html = scrape(url, scrape_params)
        if html:
            results.extend(extract(html, url))
        if page < args.pages:
            time.sleep(1)

    output = json.dumps(results, indent=2)
    if args.output:
        with open(args.output, "w") as f:
            f.write(output)
        print(f"Saved {len(results)} results to {args.output}", file=sys.stderr)
    else:
        print(output)


if __name__ == "__main__":
    main()
```

---

## Python — Async Batch (> 100 URLs)

Use when scraping a large list of known URLs. Submits all jobs at once, then polls until complete.

```python
# Requirements: pip install requests
# Usage: SCRAPERAPI_API_KEY=your_key python scraper_async.py --input urls.txt --output results.json

import os
import sys
import time
import json
import argparse
import requests

API_KEY = os.environ.get("SCRAPERAPI_API_KEY")
if not API_KEY:
    sys.exit("Error: SCRAPERAPI_API_KEY environment variable not set")

SUBMIT_URL = "https://async.scraperapi.com/batchjobs"
STATUS_URL = "https://async.scraperapi.com/jobs/{job_id}"


def submit_batch(urls: list[str], api_params: dict) -> list[dict]:
    payload = {"apiKey": API_KEY, "urls": urls, "apiParams": api_params}
    r = requests.post(SUBMIT_URL, json=payload, timeout=30)
    r.raise_for_status()
    return r.json()  # List of {id, status, url}


def poll_job(job_id: str) -> dict:
    r = requests.get(STATUS_URL.format(job_id=job_id), timeout=30)
    r.raise_for_status()
    return r.json()


def wait_for_batch(jobs: list[dict], poll_interval: int = 5) -> list[dict]:
    pending = {job["id"]: job["url"] for job in jobs}
    results = []

    while pending:
        print(f"  {len(pending)} jobs still running...", file=sys.stderr)
        time.sleep(poll_interval)
        done = []
        for job_id in list(pending):
            status = poll_job(job_id)
            if status["status"] == "finished":
                results.append({"url": pending[job_id], "html": status.get("response", {}).get("body", "")})
                done.append(job_id)
            elif status["status"] == "failed":
                print(f"  Job {job_id} failed: {pending[job_id]}", file=sys.stderr)
                done.append(job_id)
        for job_id in done:
            del pending[job_id]

    return results


def main():
    parser = argparse.ArgumentParser(description="Async batch scraper powered by ScraperAPI")
    parser.add_argument("--input", required=True, help="Text file with one URL per line")
    parser.add_argument("--output", required=True, help="Output JSON file")
    parser.add_argument("--render", action="store_true")
    parser.add_argument("--premium", action="store_true")
    parser.add_argument("--country-code")
    args = parser.parse_args()

    with open(args.input) as f:
        urls = [line.strip() for line in f if line.strip()]

    api_params: dict = {}
    if args.render:
        api_params["render"] = True
    if args.premium:
        api_params["premium"] = True
    if args.country_code:
        api_params["country_code"] = args.country_code

    print(f"Submitting {len(urls)} URLs...", file=sys.stderr)
    jobs = submit_batch(urls, api_params)
    print(f"Submitted. Polling for completion...", file=sys.stderr)
    results = wait_for_batch(jobs)

    # TODO: Add your extraction logic here — results[i]["html"] contains the raw HTML

    with open(args.output, "w") as f:
        json.dump(results, f, indent=2)
    print(f"Done. {len(results)} results saved to {args.output}", file=sys.stderr)


if __name__ == "__main__":
    main()
```

---

## Node.js — Sync (< 100 URLs)

```javascript
// Requirements: npm install node-fetch@2 commander
// Usage: SCRAPERAPI_API_KEY=your_key node scraper.js --url "https://example.com" --pages 10

const fetch = require('node-fetch');
const { program } = require('commander');
const fs = require('fs');

const API_KEY = process.env.SCRAPERAPI_API_KEY;
if (!API_KEY) { console.error('Error: SCRAPERAPI_API_KEY not set'); process.exit(1); }

const BASE_URL = 'https://api.scraperapi.com/';

async function scrape(url, params, attempt = 0) {
  const qs = new URLSearchParams({ api_key: API_KEY, url, ...params });
  try {
    const res = await fetch(`${BASE_URL}?${qs}`, { timeout: 60000 });
    if (res.ok) return res.text();
    if (res.status === 401) { console.error('Error: Invalid API key'); process.exit(1); }
    if (res.status === 403) throw new Error('Blocked (403). Try --premium or --ultra-premium.');
    if (res.status === 404) { console.warn(`Not found (404): ${url}`); return null; }
    if ([429, 500, 503].includes(res.status)) {
      if (attempt < 4) {
        const wait = 1000 * (2 ** attempt);
        console.error(`Retrying in ${wait / 1000}s (status ${res.status})...`);
        await new Promise(r => setTimeout(r, wait));
        return scrape(url, params, attempt + 1);
      }
      throw new Error(`Max retries exceeded (status ${res.status})`);
    }
    throw new Error(`Unexpected status: ${res.status}`);
  } catch (err) {
    if (attempt < 4 && err.type === 'system') {
      await new Promise(r => setTimeout(r, 1000 * (2 ** attempt)));
      return scrape(url, params, attempt + 1);
    }
    throw err;
  }
}

function buildPageUrl(base, page) {
  // TODO: Adapt for the target site's pagination scheme
  return page === 1 ? base : `${base}?page=${page}`;
}

function extract(html, url) {
  // TODO: Replace with your parsing logic
  // const cheerio = require('cheerio'); const $ = cheerio.load(html);
  // return $('li.product-item').map((_, el) => ({ title: $(el).text() })).get();
  return [{ url, htmlLength: html.length }];
}

program
  .requiredOption('--url <url>', 'Target URL')
  .option('--pages <n>', 'Pages to scrape', '1')
  .option('--output <file>', 'Output file (default: stdout)')
  .option('--max-credits <n>', 'Abort if estimate exceeds this', parseInt)
  .option('--render', 'Enable JS rendering')
  .option('--premium', 'Use premium proxies')
  .option('--ultra-premium', 'Use ultra premium proxies')
  .option('--country-code <cc>', 'Proxy country')
  .option('--session <n>', 'Session number for IP stickiness', parseInt)
  .parse();

const opts = program.opts();
const pages = parseInt(opts.pages);
let costPer = 1;
if (opts.render) costPer = 10;
if (opts.ultraPremium) costPer = 30;
else if (opts.premium) costPer = Math.max(costPer, 10);
const estimated = pages * costPer;

console.error(`Estimated: ${pages} pages × ${costPer} credits = ~${estimated} credits`);
if (opts.maxCredits && estimated > opts.maxCredits) {
  console.error(`Error: Estimated ${estimated} credits exceeds --max-credits=${opts.maxCredits}`);
  process.exit(1);
}

const scrapeParams = {};
if (opts.render) scrapeParams.render = 'true';
if (opts.premium) scrapeParams.premium = 'true';
if (opts.ultraPremium) scrapeParams.ultra_premium = 'true';
if (opts.countryCode) scrapeParams.country_code = opts.countryCode;
if (opts.session) scrapeParams.session_number = String(opts.session);

(async () => {
  const results = [];
  for (let page = 1; page <= pages; page++) {
    const url = buildPageUrl(opts.url, page);
    console.error(`Scraping page ${page}/${pages}: ${url}`);
    const html = await scrape(url, scrapeParams);
    if (html) results.push(...extract(html, url));
    if (page < pages) await new Promise(r => setTimeout(r, 1000));
  }
  const output = JSON.stringify(results, null, 2);
  if (opts.output) {
    fs.writeFileSync(opts.output, output);
    console.error(`Saved ${results.length} results to ${opts.output}`);
  } else {
    console.log(output);
  }
})().catch(err => { console.error('Fatal:', err.message); process.exit(1); });
```

---

## Python — Structured Data Endpoint

Use instead of the sync/async templates when a structured vertical is available for the target site. No HTML parsing needed.

```python
# Requirements: pip install requests
# Usage: SCRAPERAPI_API_KEY=your_key python structured.py --query "laptop" --pages 3

import os
import sys
import json
import time
import argparse
import requests

API_KEY = os.environ.get("SCRAPERAPI_API_KEY")
if not API_KEY:
    sys.exit("Error: SCRAPERAPI_API_KEY environment variable not set")

# TODO: Set the correct vertical, e.g. "amazon/search", "google/search", "walmart/product"
VERTICAL = "google/search"
BASE_URL = f"https://api.scraperapi.com/structured/{VERTICAL}"


def fetch_structured(params: dict, attempt: int = 0) -> dict:
    req_params = {"api_key": API_KEY, **params}
    r = requests.get(BASE_URL, params=req_params, timeout=60)
    if r.status_code == 200:
        return r.json()
    if r.status_code == 401:
        sys.exit("Error: Invalid API key (401)")
    if r.status_code in (429, 500, 503) and attempt < 4:
        time.sleep(2 ** attempt)
        return fetch_structured(params, attempt + 1)
    r.raise_for_status()


def main():
    parser = argparse.ArgumentParser()
    # TODO: Replace --query with the required param for your vertical
    # Amazon product: --asin; Amazon search: --query; Walmart product: --product-id
    parser.add_argument("--query", required=True, help="Search query")
    parser.add_argument("--pages", type=int, default=1)
    parser.add_argument("--country-code", default="us")
    parser.add_argument("--output", help="Output file (default: stdout)")
    args = parser.parse_args()

    results = []
    for page in range(1, args.pages + 1):
        params = {"query": args.query, "country_code": args.country_code, "page": page}
        print(f"Fetching page {page}...", file=sys.stderr)
        data = fetch_structured(params)
        results.append(data)
        if page < args.pages:
            time.sleep(1)

    output = json.dumps(results, indent=2)
    if args.output:
        with open(args.output, "w") as f:
            f.write(output)
    else:
        print(output)


if __name__ == "__main__":
    main()
```
