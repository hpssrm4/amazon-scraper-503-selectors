# Amazon Scraper Python: Why `requests` Returns 503, Which Selectors Actually Work, and When a Paid API Beats Maintaining Your Own

A Python Amazon scraper that runs on your laptop this afternoon and one that still runs in six weeks are two different projects. The first is a couple hundred lines of `requests` and BeautifulSoup. The second needs proxy rotation, retry logic, headless rendering, and someone who notices when Amazon changes a class name.

The split is volume and repetition. A few hundred product pages, one marketplace, one time: write it yourself, the script below is enough. A daily price check across thousands of ASINs, or more than one Amazon domain: price out an API before you price out your own maintenance time. ScraperAPI's free plan is 1,000 credits a month with 5 concurrent threads and no card required, and Amazon requests cost **5 credits each**, so that free allowance covers 200 product pages. The 7-day trial gives you 5,000 credits to test with, also without a card.

> **Most recurring projects should start on Hobby.** 100,000 credits a month, 20 concurrent threads, US and EU geotargeting. At 5 credits per Amazon request that's 20,000 product pages monthly, enough for a daily refresh of a few hundred ASINs with room for retries. 👉 [Check what the Hobby tier includes](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Every ScraperAPI plan, side by side

ScraperAPI sizes its plans by monthly credits and concurrency. Annual billing takes 10% off every tier except Enterprise, which is quoted custom. Credits reset at renewal and do not roll over on any plan.

| Plan | Credits / month | Amazon pages at 5 credits | Concurrent threads | Geotargeting | Start here |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | 200 | 5 | Not listed | [Create a free account](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | 100,000 | 20,000 | 20 | US & EU | [Open Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 200,000 | 50 | US & EU | [Open Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 600,000 | 100 | Country-level | [Open Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 1,000,000 | 200 | Country-level | [Open Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 2,100,000 | 300 | Country-level | [Open Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 4,300,000 | 500 | Country-level | [Open Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 4,400,000+ | 500+ | Country-level | [See Enterprise terms](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Three billing rules sit outside the table and change how you should size a plan:

- **Pay as you go** exists only on Scaling, Professional, Advanced and Enterprise. It keeps requests running past your monthly limit at a fixed per-credit rate, capped at a number you set.
- **Auto-upgrade** is opt-in on Hobby through Professional. When credits run out, the account moves to the next tier instead of pausing.
- **Concurrency overflows are refused, not billed.** Exceed your thread limit and you get a 429 telling you to come back inside the plan's concurrency. The request is turned away rather than queued, and there's no extra charge for it.

Refunds are 7 days, no questions asked, which lines up with the trial window. If the credit math turns out wrong for your workload, that's your exit.

## The 80-line version you can write this afternoon

Setup is a virtual environment and four packages:

bash
python3 -m venv .env && source .env/bin/activate
pip install requests beautifulsoup4 lxml pandas


Then send the request with browser-like headers, because a bare `requests.get()` to a product URL returns a 503 and a block page instead of HTML. The widely copied walkthrough from Oxylabs uses this shape:

python
import requests
from bs4 import BeautifulSoup

headers = {
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 13_1) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/16.1 Safari/605.1.15",
    "Accept-Language": "en-GB,en;q=0.9",
}

url = "https://www.amazon.com/Bose-QuietComfort-45-Bluetooth-Canceling-Headphones/dp/B098FKXT8L"
r = requests.get(url, headers=headers)
print(r.status_code)

soup = BeautifulSoup(r.text, "lxml")
title = soup.select_one("#productTitle").text.strip()
rating = soup.select_one("#acrPopover")["title"].replace("out of 5 stars", "").strip()
price = soup.select_one("span.a-offscreen").text
image = soup.select_one("#landingImage")["src"]


Those user-agent strings are the ones the guide published; whether a given UA still passes is something you'll find out on your first run, and probably again next month. The structure around them is what matters: a headers dict you can swap, a status check before parsing, and a fallback that retries with a different UA when you get a 503.

For listing pages, every product sits in a `div` carrying a `[data-asin]` attribute, and the links live in `h2 a` inside it. Those hrefs are relative, so `urllib.parse.urljoin` against the listing URL is mandatory or every product fetch fails. Pagination is `a.s-pagination-next`; read its `href`, join it, recurse. Keep a module-level `visited_urls = set()` so a paginated crawl doesn't re-scrape the same ASIN twice.

That's the whole tutorial: headers, three or four CSS selectors, a URL join, a visited set, and `pd.DataFrame(data).to_csv("headphones.csv", index=False)` at the end.

## Why the same script returns 503 next week

Without headers, Amazon answers with a block page whose text reads "To discuss automated access to Amazon data, please contact api-services-support@amazon.com." and a status code of 503, not 200. The guide's phrasing is that Amazon knows the request didn't come from a browser.

Adding a `user-agent` fixes it often enough to be misleading. Sometimes the UA alone is enough; other times you also need `accept-language`, and the guide's second header set adds `accept-encoding`, `accept` and a `referer` from Google. Rotating several UA strings and retrying is the suggested workaround for a 503.

Notice what that workaround is: guess, retry, hope. It's fine for a one-off CSV. It falls apart when you need 5,000 pages a day, because retrying from the same IP just gets you blocked again with a different status code. Headless browsers enter here too. If a page needs JavaScript to render, Playwright or Selenium will run it, and both will happily render the block page Amazon served your datacenter IP. Rendering solves HTML, not reputation.

CAPTCHAs are the third wall. A script can detect one; solving it at volume means a solving service or a fingerprint that doesn't trigger it, and that is a full-time maintenance line item, not a function you write once.

## The parsing details tutorials skip

The selectors are where quiet bugs live, because a wrong selector doesn't throw. It returns `None`, and `None` becomes an empty column you notice two weeks later.

| Field | Selector | The part that bites |
| --- | --- | --- |
| Title | `#productTitle` | Trailing whitespace; needs `.strip()` |
| Rating | `#acrPopover` | The number sits in the `title` attribute as `4.6 out of 5 stars`, not in the text; strip the suffix |
| Price | `span.a-offscreen` | Amazon also splits price into `a-price-whole` and `a-price-fraction` spans, so a fallback path is worth having |
| Main image | `#landingImage` | Read the `src` attribute, not the text |
| Description | `#productDescription` | Only exists on product pages, not on search results |
| Listing links | `[data-asin] h2 a` | Relative hrefs; needs `urljoin` |
| Next page | `a.s-pagination-next` | Absent on the last page; guard for `None` |

Out-of-stock items are the other trap. The page still loads, the title and image parse fine, and the price field comes back empty. Oxylabs handles this in its own free Amazon scraper by leaving the `price` column blank and flagging the item as out of stock rather than dropping the row. Whatever you do, decide it explicitly, because a CSV with silent empty prices will quietly poison any average you compute from it.

## Wiring ScraperAPI into the same script

ScraperAPI sits in front of the request. You keep your parsing code; the proxies, retries, CAPTCHA handling and fingerprinting move off your machine. Amazon is priced as an e-commerce target at 5 credits per request, five times a standard page.

For Amazon specifically there's a structured endpoint that returns parsed JSON instead of HTML, so the selector table above stops being your problem. The response includes the product name, a `brand` field, a `product_information` object with `asin`, `best_sellers_rank`, dimensions, weight and `date_first_available`, an `images` array of full URLs, a `full_description`, and an `is_coupon_exists` flag. That last one is awkward to derive from HTML and useful if you're tracking promotions.

Optional parameters stack credits on top of the base 5:

| Parameter | Extra credits | When it applies |
| --- | --- | --- |
| `render=true` | +10 | Page needs JavaScript |
| `premium=true` | +10 | Residential-style proxies |
| `ultra_premium=true` | +30 | Paid plans only |
| `screenshot=true` | +10 | Visual capture |
| Bot-protection bypass | +10 | Cloudflare, DataDome, PerimeterX/HUMAN, Akamai |
| Geotargeting, sessions, device type, output format, auto-parsing | 0 | Free across plans |

Do the multiplication before you choose a tier. A plain Amazon product page is 5 credits. The same page with premium proxies and rendering is 25, so a plan sized on 5 credits per page delivers a fifth of the pages you budgeted for. Ultra premium plus rendering reaches 75 credits, fifteen times the base rate. This is the number that separates a plan that fits from one that runs dry on the 6th of the month.

Two controls exist for exactly this. The Domain Cost Estimator in the dashboard prices any URL before you send it, and every response carries an `sa-credit-cost` header showing what that request actually consumed. `max_cost` caps a single scrape so one expensive URL can't eat a day's allowance.

## What each approach actually costs

| Approach | What you pay for | Reference point |
| --- | --- | --- |
| Your own script | Proxy bandwidth, headless browser hosting, and the hours spent re-fixing selectors and block pages | No fixed number; the maintenance is the largest line and it never appears on an invoice |
| ScraperAPI | Credits per request, 5 for Amazon, plus any parameter multipliers | 1,000 credits free monthly; 100,000 on Hobby |
| Oxylabs (market reference) | Per delivered result rather than per attempt | Amazon requests from $0.50 per 1,000 results; $5 welcome credit on new accounts |

The billing model matters as much as the unit price. Oxylabs charges for successfully delivered results, so a blocked attempt costs nothing. ScraperAPI charges credits per request, which is why the `sa-credit-cost` header and the cost estimator are worth two minutes of your time before committing to a volume. If most of your Amazon requests need premium proxies plus rendering, your effective cost per page is five times the headline rate, and a plan that looks oversized on credits may be exactly right.

## When you shouldn't buy credits at all

Plenty of Amazon scraping jobs don't justify a subscription:

- A one-off pull of a few dozen ASINs for a spreadsheet. Run the script above, fix the two or three selectors that break, done.
- A class assignment or a portfolio project where nobody cares if it runs again next month.
- Anything you can get from a manual export or Seller Central reports.
- A single marketplace, single category, refreshed weekly, under a few hundred products, where you're willing to babysit a 503 now and then.

The free plan covers the testing phase for these: 200 Amazon pages a month is enough to confirm your parsing logic against real JSON before you spend anything.

## Picking a plan, and two things to check before you pay

Run your own numbers against these rules:

- **Under 200 Amazon pages a month, one-off or experimental** → free plan, 1,000 credits.
- **A few hundred ASINs refreshed daily** → Hobby. 20,000 Amazon pages a month leaves headroom for retries and the occasional rendered page.
- **One full category crawl plus a competitor set, weekly** → Startup at 1,000,000 credits, and check whether country-level geotargeting matters to you, because it starts at Business.
- **Multiple marketplaces with delivery-location pricing** → Business or above; country-level targeting and 100 threads are the reasons.
- **Volume where running dry mid-month is unacceptable** → Scaling or higher, where pay as you go exists as a safety valve.

Before you enter a card, confirm two things. First, what a credit actually costs for *your* URLs: put a handful of your real Amazon URLs through the Domain Cost Estimator, then watch the `sa-credit-cost` header on the first live responses, and set `max_cost` so a single heavy page can't blow the budget. Second, what happens at the limits: credits don't roll over, concurrency overflows return a 429 instead of billing you for the overage, and the refund window is 7 days from purchase.

If both answers match your workload, the trial costs nothing to run against your actual target list. 👉 [Start the 7-day trial with 5,000 credits](https://www.scraperapi.com/?fp_ref=coupons) and point it at the ASINs you already know the answers for.
