# crawlee proxy: Wiring rotating residential proxies into Crawlee with working session pinning, realistic cost-per-1,000-pages math, and the right proxy tier for each target

Most people searching for Crawlee proxy setup already know the shape of the answer: build a proxy URL, hand it to the crawler, move on. The part that actually costs money is what happens after the first `403`. A Crawlee crawler with a badly pinned session burns bandwidth on retries, and retries are billed per GB just like successful requests. So the setup question and the budget question are the same question.

Below: where proxies plug into Crawlee in both JavaScript and Python, why session handling decides your block rate more than the IP pool does, how to run DataImpulse endpoints inside Crawlee, and what all of it costs at different scales.

## Crawlee will rotate proxies. It will not make them trustworthy.

Crawlee ships a `ProxyConfiguration` class and a `SessionPool` that are already wired together. If you use them correctly, proxy rotation happens automatically, including retiring a browser instance after a set number of requests so a fresh IP gets attached to the next one. That's real functionality, and it's the reason Crawlee is a reasonable starting point for a scraper instead of a hand-rolled loop.

What Crawlee cannot fix is the reputation of the IPs you feed it.

Blocks against scrapers usually trace back to three things: the IP's history, the request rate from that IP, and browser fingerprints. Crawlee handles fingerprinting reasonably well out of the box. The other two are on your proxy provider. Datacenter IPs from a shared cloud range get flagged by reputation scoring before your request even reaches the page logic, which is why a crawler that runs fine on one target returns nothing but challenge pages on another.

That's the whole reason residential proxies show up in every Crawlee tutorial: the IP belongs to a real broadband connection, so anti-bot systems have to judge it on behavior instead.

## Where proxies plug into Crawlee

### Passing proxy URLs

The simplest form is a static list. Crawlee takes `proxyUrls` and assigns them to requests. In JavaScript:

javascript
import { CheerioCrawler, ProxyConfiguration } from 'crawlee';

const proxyConfiguration = new ProxyConfiguration({
    proxyUrls: [
        'http://login__cr.us:password@gw.dataimpulse.com:823',
        'http://login__cr.de:password@gw.dataimpulse.com:823',
    ],
});

const crawler = new CheerioCrawler({
    proxyConfiguration,
    maxRequestRetries: 3,
    async requestHandler({ request, body, log }) {
        log.info(`${request.url} fetched with ${request.proxyInfo?.url ?? 'no proxy'}`);
    },
});

await crawler.run(['https://example.com/page/1']);


The Python crawlers follow the same pattern with snake_case arguments:

python
import asyncio
from crawlee.beautifulsoup_crawler import BeautifulSoupCrawler, BeautifulSoupCrawlingContext
from crawlee.proxy_configuration import ProxyConfiguration

async def main() -> None:
    proxy_configuration = ProxyConfiguration(
        proxy_urls=["http://login__cr.us:password@gw.dataimpulse.com:823"],
    )

    crawler = BeautifulSoupCrawler(
        proxy_configuration=proxy_configuration,
        max_request_retries=3,
        max_concurrency=10,
    )

    @crawler.router.default_handler
    async def handler(context: BeautifulSoupCrawlingContext) -> None:
        status = context.http_response.status_code
        if status in (403, 429):
            context.session.retire()
            raise RuntimeError(f"blocked with {status}")

        title = context.soup.find("title")
        await context.push_data({
            "url": context.request.url,
            "title": title.text.strip() if title else None,
        })

    await crawler.run(["https://example.com/page/1"])

asyncio.run(main())


Two details in that Python snippet matter more than they look. `context.session.retire()` drops the session, which in Crawlee means dropping the cookies, the browser state, and the proxy it was bound to. The retry then runs on a new session with a new IP. And `max_request_retries=3` caps how many times you pay for that attempt.

### Session pinning is the part that decides your block rate

Crawlee's `ProxyConfiguration` can generate a proxy per request or bind a proxy to a session ID. If you go with per-request rotation, every request looks like it comes from a different place. That breaks logins, paginated lists, and anything that expects continuity.

Session pinning means the same session keeps the same IP across multiple requests, so cookies, IP, and fingerprint stay consistent. From the scaper's side you appear to be one user browsing a few pages. That's the pattern that keeps multi-step flows working, and it's why providers expose session tokens in the proxy username rather than asking you to manage IP lists.

With DataImpulse, targeting and session parameters go directly in the username field, using double underscores as a delimiter, a dot between key and values, and semicolons between different parameters. Country selection looks like this:


http://login__cr.de:password@gw.dataimpulse.com:823


Country and session are configured either in the dashboard's proxy configuration panel or by appending the parameters manually, and Crawlee will pick up whatever URL you supply. That means the rotation strategy lives in the provider's username string, not in custom Crawlee code.

## What Crawlee will not do for you

Three gaps show up repeatedly:

**It doesn't throttle by target, only by pool.** Concurrency limits in Crawlee are global to the crawler. If you're crawling one site politely and another aggressively, that's on your configuration, not Crawlee's defaults.

**It doesn't check `robots.txt`.** Crawlee won't stop you from hitting disallowed paths. That stays your decision, and it's a short conversation in most projects.

**It doesn't tell you when retries are eating your budget.** Every retry that returns another challenge page is billed traffic with zero data extracted. On a pay-per-GB plan, a crawler with sloppy block detection can cost several times what a clean run would.

## Which proxy type belongs in your Crawlee crawler

Not every crawl needs residential IPs, and paying residential rates for unprotected targets is a straightforward waste. The type of target decides the tier:

| Target type | Sensible proxy tier | Why |
| --- | --- | --- |
| Static sites, sitemaps, docs, no anti-bot layer | Datacenter | Cheapest per GB, high uptime, nothing to defeat |
| E-commerce, SERPs, marketplaces, price monitoring | Residential | IP reputation is the blocker, not request logic |
| Cloudflare/Datadome-protected pages, app-like mobile web | Mobile | Carrier-grade NAT makes the IP very hard to block outright |
| High-value targets where a failed request costs more than traffic | Premium residential | Highest-trust pool, targeting included at no extra charge |

The general rule: start with the cheapest tier that returns real data. If your Crawlee logs are full of `403` and challenge-page titles, move up a tier rather than pushing concurrency higher.

If you're deciding where to start, the 👉 [DataImpulse pay-as-you-go proxy plans](https://bit.ly/dataimPulse) let you buy a few GB, watch what your crawler actually consumes, and scale only after the cost per successful request makes sense against the dataset you need.

## The bandwidth math nobody runs before buying

Per-GB pricing hides the real number, which is cost per usable record. Working it out with one simplified example is more useful than reading competitor tables.

Take a Cheerio-based crawl of 1,000 list pages, averaging roughly 200 KB of transferred HTML per request, gzip included. That's about 200 MB, or roughly 0.2 GB. At $1/GB for residential traffic, the raw cost is around $0.20. Add a 20% retry rate caused by intermittent blocks and you're at about $0.24.

Now swap in a `PlaywrightCrawler` doing the same job. A browser instance pulls the page plus every asset the page requests. A single navigation can easily move 1–3 MB. Same 1,000 pages, and you're looking at 1–3 GB instead of 0.2 GB, so $1 to $3 before retries. That's a 5× to 15× difference on an identical task.

The practical conclusion for Crawlee users: reach for the browser crawler only when the page genuinely requires JavaScript execution. SERP tracking, price monitoring, and catalog extraction usually don't. Many SPAs fetch their data from a JSON endpoint you can call directly, which is worth checking in DevTools before committing to browser crawling.

## Hooking DataImpulse endpoints into Crawlee

DataImpulse runs a single gateway rather than per-country endpoints. The host is `gw.dataimpulse.com`, with port `823` for HTTP/HTTPS and port `824` for SOCKS5. Geographic and session parameters are appended to the username, so one set of credentials covers all countries.

The parameters can be set from the dashboard when you create an endpoint, or typed manually into the username. Manual format:


key1.value1,value2;key2.value1,value2


So a Germany-targeted endpoint is `login__cr.de`, and a two-country endpoint is `login__cr.de,au`.

A few things worth knowing before you point a large crawler at it:

- **Country targeting is included in the base price.** That's the level Crawlee requests usually need.
- **City, state, ZIP, and ASN targeting costs double** on standard residential plans. If your Crawlee config pins a city, budget accordingly. Datacenter proxies list state/city/ZIP/ASN targeting as included, though it's worth confirming the current billing treatment with support before you build a budget around it.
- **Concurrency has a ceiling.** Accounts hitting more than 2,000 active connections get a `407 THREADS_EXHAUSTED` response. Keep `max_concurrency` well under that unless you've confirmed a higher limit.
- **Requesting a location with no matching IP returns `503 NO_RAY`.** In practice this means your targeting is too narrow, usually a city filter. Drop to country level and it resolves.
- **Bought traffic doesn't expire.** A GB bought this month is still there in six months, which matters a lot for crawlers that run in bursts rather than continuously.
- **New users have a 7-day refund window**, which is enough time to run a realistic crawl and check the success rate against your own targets rather than someone else's benchmark.

The provider advertises 99.51% success rate and a 4.8/5 G2 rating, along with a pool of 90M+ IPs across 195 countries. Treat those as vendor figures and validate with your own crawl; a proxy pool's success rate is target-specific, and no advertised number survives contact with a hostile site.

## Error mapping: what each failure actually means

When a Crawlee run goes wrong, the response code tells you which layer failed. Getting this wrong is how people end up buying additional traffic they didn't need.

| What you see | Where the problem is | Sensible move |
| --- | --- | --- |
| `403` on target | Target's anti-bot layer rejected the IP | Switch country or retire the session once; don't loop |
| `429` on target | Rate limiting on that IP | Lower concurrency, lengthen delays, keep session pinning |
| `407 TRAFFIC_EXHAUSTED` | Account balance is empty | Top up; retries won't fix it |
| `407 THREADS_EXHAUSTED` | Over 2,000 concurrent connections | Reduce `max_concurrency` |
| `503 NO_RAY` | No IP matches the targeting | Remove city/ASN filters, keep country only |
| Timeouts, no status | Endpoint or protocol mismatch | Check port `823` (HTTP/HTTPS) vs `824` (SOCKS5) |

In Crawlee terms, the cleanest pattern is to detect `403` and `429` inside the request handler, retire the session, and rethrow so Crawlee's retry logic handles it with exponential backoff. Crawlee's first retry waits around a second, the second about two, the third about four. That gives the newly assigned IP time to take effect instead of hammering the same block.

Keep `maxRequestRetries` in the 3–5 range. Higher values on a pay-per-GB plan mostly buy you repeated blocks at full price.

## Full plan and price comparison

DataImpulse sells four proxy types, all pay-as-you-go with no subscription. Below are the packages currently listed publicly, with volume tiers where they exist. Residential and datacenter are the tiers a typical Crawlee deployment actually uses; mobile and premium residential exist for hostile targets where the per-GB cost is not the deciding factor.

| Proxy type | Package | Traffic | Effective rate | Billing | Buy |
| --- | --- | --- | --- | --- | --- |
| Residential | Starter | 5 GB | $1.00/GB | Pay-as-you-go, traffic never expires | [Get the 5 GB starter](https://bit.ly/dataimPulse) |
| Residential | Standard | 50 GB | $1.00/GB | Pay-as-you-go | [Get 50 GB](https://bit.ly/dataimPulse) |
| Residential | Volume | 1 TB | $0.80/GB ($800) | Pay-as-you-go | [Get the 1 TB volume tier](https://bit.ly/dataimPulse) |
| Residential | Bulk | 5 TB | ~$0.70/GB | Pay-as-you-go, volume discount | [Check 5 TB pricing](https://bit.ly/dataimPulse) |
| Datacenter | Starter | 10 GB | $0.50/GB | Pay-as-you-go, 99.9% uptime | [Get 10 GB datacenter](https://bit.ly/dataimPulse) |
| Datacenter | Standard | 100 GB | $0.50/GB ($50) | Pay-as-you-go | [Get 100 GB datacenter](https://bit.ly/dataimPulse) |
| Datacenter | Volume | 1 TB | $0.45/GB ($450) | Pay-as-you-go | [Get 1 TB datacenter](https://bit.ly/dataimPulse) |
| Datacenter | Enterprise | 5 TB+ | Custom, from $2,250 | Contract, subnets included | [Request datacenter enterprise pricing](https://bit.ly/dataimPulse) |
| Mobile | Starter | 2.5 GB | $2.00/GB | Pay-as-you-go, 4G/5G/LTE | [Get 2.5 GB mobile](https://bit.ly/dataimPulse) |
| Mobile | Standard | 25 GB | $2.00/GB ($50) | Pay-as-you-go | [Get 25 GB mobile](https://bit.ly/dataimPulse) |
| Mobile | Volume | 1 TB | $1.60/GB ($1,600) | Pay-as-you-go | [Get 1 TB mobile](https://bit.ly/dataimPulse) |
| Mobile | Enterprise | 5 TB+ | Custom, from $8,000 | Contract | [Request mobile enterprise pricing](https://bit.ly/dataimPulse) |
| Premium residential | Starter | 1 GB | $5.00/GB | Pay-as-you-go, all targeting included | [Get 1 GB premium residential](https://bit.ly/dataimPulse) |
| Premium residential | Standard | 10 GB | $5.00/GB ($50) | Pay-as-you-go, dedicated account manager | [Get 10 GB premium residential](https://bit.ly/dataimPulse) |
| Premium residential | Enterprise | 5 TB+ | Custom, from $20,000 | Contract, high-speed pool | [Request premium enterprise pricing](https://bit.ly/dataimPulse) |

All four types support HTTP(S) and SOCKS5, rotating and sticky sessions, and country-level targeting in the base rate. Residential traffic applies a 2× multiplier when routed through state, city, ZIP, or ASN filters, so a Crawlee job that pins cities on residential effectively runs at $2/GB.

## FAQ

**Does Crawlee work with SOCKS5 proxies?**
The Python crawlers accept SOCKS proxy URLs through `ProxyConfiguration`. In JavaScript, browser-based crawlers pass the proxy to the underlying browser, and SOCKS5 support depends on that layer. HTTP(S) on port `823` is the safer default for JS.

**Should I rotate the IP on every request?**
Only for stateless jobs where each page stands alone. Paginated crawls, logged-in flows, and anything with cart state need session pinning, otherwise you'll trigger more challenges than you avoid.

**How many concurrent requests can I run?**
Start at `max_concurrency` 10 to 20. DataImpulse caps accounts at 2,000 active connections, and residential pools behave better under moderate load than under a few hundred simultaneous requests from one session set.

**What's the cheapest setup for a Crawlee crawler that runs a few times a month?**
Datacenter for anything unprotected, residential for defended targets, both pay-as-you-go. Traffic doesn't expire, so the burst pattern doesn't waste purchased GB the way a monthly subscription would. 👉 [Start with a $5 residential top-up](https://bit.ly/dataimPulse) and measure your own cost per successful request before scaling.

**Do I need the premium residential tier?**
Only if your targets are actively hostile and you're already using all targeting options. It includes every targeting filter at no surcharge and a dedicated account manager, which is a different purchasing logic than a $1/GB scraping job.
