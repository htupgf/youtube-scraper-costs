# youtube scraper: How to Pull Video Data Without Getting Blocked, and What It Actually Costs per Request

Scraping YouTube comes down to three data sources: the official YouTube Data API, free open-source tools like yt-dlp, and a scraping API that routes your requests through a rotating proxy pool. The official API is reliable but quota-capped, open-source tools are free but fragile at scale, and a scraping API like ScraperAPI handles the blocking for you at a per-request credit cost. This article walks through what a youtube scraper actually needs to handle, compares the options, and shows the real math behind ScraperAPI's plans — including a number most pricing pages won't tell you: how few requests your credits really buy once JavaScript rendering kicks in.

> **Quick pick:** Most people testing a youtube scraper should start with the free trial — 5,000 API credits for 7 days, no credit card required. If you're running a small ongoing project, Hobby at $49/month (or $44.10/month billed annually) covers roughly 9,000 rendered YouTube page requests per month. 👉 [Start the 7-day free trial with 5,000 credits](https://www.scraperapi.com/signup?fp_ref=coupons)

## Why YouTube scrapers break, and what your options really are

YouTube is one of the hardest mainstream sites to scrape, and the reason is structural. Video pages load almost everything through JavaScript, so a plain HTTP request gets you a shell, not data. On top of that, YouTube fingerprints browser headers and session patterns, rate-limits at the routing layer, and serves different content to different regions — which is why a scraper that works locally can be returning 429s and empty HTML within two days of going live.

Your three realistic ways around this, in ascending order of effort-to-reliability tradeoffs:

**yt-dlp and free open-source scripts.** yt-dlp extracts metadata (title, duration, channel, like counts), pulls comments with `{"getcomments": True}`, and downloads video files — all free, no API key. A typical setup needs Python 3.8+ and one install command: `pip install yt-dlp requests beautifulsoup4`. GitHub also hosts several open-source youtube scraper projects that parse data straight out of YouTube's embedded `ytInitialPlayerResponse` JSON without a browser. The catch is durability: when YouTube changes something, you're the one fixing the parser. Fine for a one-off export, risky as a scheduled pipeline.

**Marketplace scraper actors.** Platforms like Apify host ready-made YouTube actors that return structured JSON and charge per result or per compute unit. You don't maintain anything, but your data schema is whatever the actor author decided, and costs scale with volume in ways that are hard to predict before you run the job.

**A general scraping API.** This is where ScraperAPI sits. Instead of a YouTube-specific product, you send any YouTube URL to its API endpoint with your key, and the service returns fully rendered HTML from a pool of 40M+ proxies across 50+ geolocations. You parse it yourself (the metadata lives in `ytInitialPlayerResponse`, no headless browser needed), so your schema never breaks because someone else's actor got deprecated.

On reliability, keep two numbers separate. ScraperAPI's own product page claims a near 100% success rate for its Async Scraper and advertises a custom-trial tier at 99.99% on YouTube — those are vendor figures. The independent datapoint comes from a third-party writeup citing Proxyway's 2025 scraper API benchmark, which put ScraperAPI's YouTube unblocking success rate at 94.48% at 2 requests per second. That trails Oxylabs and Zyte at 100%, but the same source notes ScraperAPI is significantly cheaper than those enterprise-priced alternatives. Treat the gap as the price-of-proxy-quality question and test against your own targets.

If you only need one video's metadata once a week, stop reading and use yt-dlp. The paid options only make sense when you're collecting at scale or on a schedule.

## ScraperAPI plans and pricing

Every paid plan includes the same feature set: JS rendering, premium residential and mobile IPs, CAPTCHA and anti-bot handling, JSON auto-parsing, DataPipeline access, and unlimited bandwidth. The plans differ in exactly four things: credits, concurrent threads, geotargeting scope, and analytics dashboard retention.

| Plan | Monthly price | Annual (10% off) | Credits/month | Concurrent threads | Geotargeting |
| --- | --- | --- | --- | --- | --- |
| Hobby | $49 | $44.10/mo | 100,000 | 20 | US & EU only |
| Startup | $149 | $134.10/mo | 1,000,000 | 50 | US & EU only |
| Business | $299 | $269.10/mo | 3,000,000 | 100 | Country-level |
| Scaling (most popular) | $475 | $427.50/mo | 5,000,000 | 200 | Country-level |
| Professional | $975 | $877.50/mo | 10,500,000 | 300 | Country-level |
| Advanced | $1,975 | $1,777.50/mo | 21,500,000 | 500 | Country-level |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ | Country-level |

Two things to know before comparing rows. Annual billing saves 10% across every tier. And the two free ways in are different mechanisms: the **free trial** gives 5,000 one-time credits over 7 days with no credit card, while the **free plan** is permanent but much smaller — 1,000 credits and 5 concurrent connections. Don't conflate them when budgeting your first month.

Going over your concurrency limit costs nothing on any plan — you get a 429 telling you to slow down, and the request is refused rather than queued.

## The math most reviews skip: what a YouTube request really costs

Here's the part that changes which plan you should buy. ScraperAPI bills in credits, and a standard page costs 1 credit. But YouTube is a heavily client-side-rendered site, so for real youtube scraper work you'll almost always set `render=true`, which adds 10 credits per request. Your realistic base cost is **11 credits per YouTube page**.

On a Hobby plan with 100,000 monthly credits, that's about **9,000 rendered YouTube page requests per month** — a very different number from the 100,000 in the headline. Someone scraping 500 channels a week at a few pages per channel will burn through Hobby faster than the credit count suggests. Startup (1M credits ≈ 90,000 rendered requests) is the first tier that supports genuine ongoing collection.

Other cost rules from the docs that affect your budget:

- Only successful requests are billed — status codes `200` and `404`. Failed attempts don't eat your credits, but if you cancel a request yourself inside 70 seconds, it still counts as used.
- `premium=true` adds 10 credits, `ultra_premium=true` adds 30. Combined with rendering: premium + render is 25 credits total, ultra premium + render is 75.
- Parameters like `country_code`, `session_number`, `device_type`, and `output_format` cost nothing extra — geotargeting in particular is included on all plans, which matters because YouTube search results differ by region.
- Every response includes a `sa-credit-cost` header showing exactly what that request consumed, so you can verify costs in real time rather than reconciling at the end of the month.
- You can check the exact credit cost of any URL before running anything, through the Domain Multiplier in the dashboard, the API Playground, or a dedicated account endpoint that accepts all the same parameters. Setting a `max_cost` per scrape caps what any single request can consume.

That last point is the cheapest insurance available: run the cost check once before committing to a plan, and you'll know your real monthly burn instead of guessing.

## Building the scraper: yt-dlp plus ScraperAPI

The two tools combine well. yt-dlp handles anything that doesn't need unblocking — downloads, single-video metadata, comment pulls. ScraperAPI handles the requests YouTube would otherwise throttle: channel pages, search results, scheduled collection.

For ScraperAPI calls, the core pattern is a GET request with your key, the target URL, and parameters:

python
import requests

params = {
    'api_key': 'YOUR_API_KEY',
    'url': 'https://www.youtube.com/watch?v=VIDEO_ID',
    'country_code': 'us',
    'output_format': 'markdown',
    'render': 'true'
}
response = requests.get('https://api.scraperapi.com/', params=params)


For channel data, the official tutorial's pattern is the same request with `render: 'true'`, then BeautifulSoup to pull the channel name from the `ytd-channel-name` element and the description from the channel tagline block. The data you can get this way covers video metadata, search results, channel stats (subscribers, total views, video count), comments and reply threads, and auto-generated or manual transcripts. Comments are the hardest of the lot — YouTube lazy-loads them and throttles hard — so if comments are your main target, plan more credits and lower request rates.

For scheduled jobs, you can loop with `time.sleep()`, or drop ScraperAPI into its async endpoint to parallelize throughput without occupying your plan's concurrent connections.

Three production features sit above the basic endpoint:

- **LLM-ready output.** Set `output` to `text` or `markdown` and YouTube pages come back as clean text instead of raw HTML — useful for feeding video content into a chat agent, tracking audience sentiment, or building training data.
- **Async Scraper.** A POST-based service for million-request jobs: data returns via webhook, and timeout, retry, and anti-bot handling are managed for you.
- **DataPipeline.** Submit a list of YouTube URLs, choose how you want the data delivered, and the pipeline runs and schedules the whole collection without code — useful when scraping itself isn't your team's job.

## Limits and quirks buried in the docs

Several things only appear in documentation or billing pages, but they directly affect which plan fits:

1. **Geotargeting scope.** Hobby and Startup are limited to US & EU locations. If your youtube scraper needs localized search results from other countries — say, tracking how a video ranks in Japan or Brazil — Business ($299) is the first tier with full country-level targeting.
2. **Analytics retention.** Hobby and Startup dashboards keep only 30 days of request analytics. Business and above get unlimited history, which matters if you audit usage over quarters.
3. **Auto-upgrade is opt-in.** Available on Hobby through Professional: when credits run out, your account moves to the next tier automatically. Useful, but it means a runaway script can silently escalate your bill if you enable it and don't watch usage.
4. **Pay-as-you-go overflow only exists on Scaling and above.** Hobby, Startup, and Business users who exceed their credits can't keep scraping at a fixed per-credit rate — they either upgrade, wait for renewal, or rely on auto-upgrade.

If any of those four constrain your use case, they're better tiebreakers than raw credit volume.

## FAQ

**Is scraping YouTube legal?** The landscape around scraping publicly available data varies by jurisdiction and use case, and I can't give legal advice. What's verifiable here: scraping at volume means operating against YouTube's anti-bot systems, which is exactly the problem services like ScraperAPI market themselves as solving. Check your own use case and local law.

**How many credits does a YouTube request use without rendering?** One credit for a standard request. But without JS rendering you'll mostly get a page shell rather than usable data, so plan around the 11-credit rendered figure.

**Can I try it without paying?** Yes — the trial gives 5,000 credits over 7 days with no credit card, and a permanent free plan exists with 1,000 credits and 5 concurrent connections. Use the API Playground to test your target URLs before committing.

**Does it return JSON instead of HTML?** It returns the page HTML (or markdown/text via `output_format`), and the structured data lives in the page's embedded JSON, which you parse out.

**Does exceeding my credits stop my project mid-month?** On Hobby through Business, yes — your options are upgrading or waiting for renewal unless you enabled auto-upgrade. Scaling and higher plans can keep running on pay-as-you-go credits up to a monthly spending cap you set.

## Picking a plan before you sign up

Match the plan to your actual request pattern:

- **Just experimenting** → free trial (5,000 credits) or the free plan (1,000 credits, 5 concurrent). Enough for a few hundred rendered pages.
- **Ongoing small project, US/EU data only** → Hobby at $49/month. Know that it's ~9,000 rendered requests, not 100,000. 👉 [Get the Hobby plan at $49/month, $44.10 billed annually](https://www.scraperapi.com/signup?fp_ref=coupons)
- **Localized search data from any country, or analytics history you need to audit** → Business at $299/month is the floor for both.
- **Hundreds of channels tracked weekly, or comments at volume** → Scaling at $475/month, which is also the first tier with pay-as-you-go overflow so a heavy week doesn't stop your pipeline.
- **Millions of requests monthly** → Professional, Advanced, or a custom Enterprise deal with dedicated Slack support.

One last check before checkout: run your two or three most representative URLs through the cost estimator, confirm the per-request credit figure with `render=true` on, and multiply by your monthly volume. If the number fits comfortably under 85% of the plan's credits, you're buying the right tier — and if it doesn't, drop down a tier or shrink the scope before the first bill lands. 👉 [Check current plans and annual pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons)
