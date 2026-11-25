# eBay Product Scraper: Which Fields Do You Actually Need, Do You Have to Build One, and Which Plan Covers Your Volume? (Field Checklist + Plan Comparison)

If you need eBay item data on a schedule, there are three routes: write your own scraper, call a scraping API that hands back parsed eBay JSON, or rent a hosted actor that somebody else maintains. For most people pulling a few thousand listings a month, the second route wins on time. ScraperAPI runs a structured eBay product endpoint and an async eBay search endpoint, and its free plan gives you **1,000 credits and 5 concurrent connections** to test both against your own URLs before you pay anything.

What follows is the stuff that decides whether a tool is usable: the exact fields that matter, what ScraperAPI's eBay endpoints return, how credits get eaten on a target as defended as eBay, and what the self-built route costs you in code. Paid-tier allowances and monthly prices live on the pricing page and change with promotions, so the plan table below tracks what each tier does differently when your credits run out.

> **Quick pick**
> - **Validating output, or under a few thousand pages a month:** start on the free plan. 1,000 credits and 5 concurrent connections is a real test, not a demo.
> - **Recurring price or inventory pulls:** the lowest paid tier whose allowance survives your Domain Cost Estimator math, with auto-upgrade switched on so a run never stalls mid-week.
> - **Volume that spikes:** Scaling or Professional, the tiers where pay-as-you-go overflow sits alongside auto-upgrade.
> 👉 [Compare ScraperAPI plans and credit allowances](https://www.scraperapi.com/pricing?fp_ref=coupons)

## The plans, and what actually separates them

ScraperAPI lists eight tiers: Free, Hobby, Startup, Business, Scaling, Professional, Advanced and Enterprise. Credit allowance and monthly price differ per tier and are published on the pricing page. The difference you feel day to day is how each tier behaves once you hit the ceiling. By default, scraping **pauses until your credits renew**. Auto-upgrade moves you to the next tier automatically and is available on Hobby, Startup, Business, Scaling and Professional. Pay-as-you-go keeps you on your current plan at a fixed per-credit rate, capped at a monthly spend limit you set, and is available on Scaling, Professional, Advanced and Enterprise. Annual plans can also be renewed early to pull the next batch of credits forward.

| Plan | What's different | Billing | Fits |
| --- | --- | --- | --- |
| **Free** | **1,000 credits**, max **5 concurrent connections** | Free | Proving the endpoint returns usable data for your categories |
| **Hobby** | Entry paid tier; auto-upgrade available | Monthly or annual | Steady, predictable pulls where overage is rare |
| **Startup** | Auto-upgrade available | Monthly or annual | Same pattern, more headroom |
| **Business** | Auto-upgrade available | Monthly or annual | Recurring catalogue and price jobs |
| **Scaling** | Auto-upgrade **and** pay-as-you-go with a spend cap | Monthly or annual | Workloads with seasonal peaks |
| **Professional** | Auto-upgrade **and** pay-as-you-go with a spend cap | Monthly or annual | Heavier recurring jobs that occasionally burst |
| **Advanced** | Pay-as-you-go with a spend cap | Monthly or annual | High volume where you'd rather cap spend than change tier |
| **Enterprise** | Pay-as-you-go with a spend cap | Monthly or annual | Large operations that need a contract shape |

The fit column is about behaviour, not page counts, because the allowance number is the part that changes. Two more things on that page matter for eBay work: unlimited bandwidth and a 99.9% uptime guarantee are listed among the platform features, which means a big image-heavy category page costs you the same bandwidth as a thin one.

## What a usable eBay product scraper has to return

A tool is only as good as the fields it can hand you without a second pass. The published sample response from ScraperAPI's eBay product endpoint covers most of the checklist that price-monitoring and catalogue teams actually build on:

| What you need | Why it matters | Covered in the eBay product endpoint |
| --- | --- | --- |
| Item identity | Deduplication across runs | `product_id_epid`, `title` |
| Price with currency | Multi-site comparison without string parsing | `price.value` (149.99) and `price.currency` (USD) as separate fields |
| Seller quality | Filtering out low-feedback sellers changes your averages | `seller.name`, `seller_review` ("99.4% positive"), `seller_reviews_count`, `top_rated`, `seller_url` |
| Demand signals | Sold counts separate real sellers from stale listings | `sold_items` ("1,866 sold"), `available_quantity` ("More than 10 available") |
| Condition and identifiers | Feeding a catalogue needs UPC/MPN, not just a title | `condition` ("Excellent - Refurbished"), `brand`, `model`, `year_manufactured`, `color`, and `item_specifics` as label/value pairs including UPC and MPN |
| Shipping and returns | Landed cost and return windows drive margin math | `shipping_costs`, `shipping_notes`, `return_policy` |
| Images | Listing quality and visual matching | `images` array, multiple sizes per item |

Two things worth noting from that sample. The `item_specifics` array is where the messy data lives, and it keeps labels intact ("Seller Notes", "MPN", "Form Factor"), so you get the seller's own wording rather than a guessed schema. And the sample response is item-level. If your use case lives on variant-level price and stock, size or colour combinations with separate quantities, test two or three multi-variant listings on the free plan before you commit to a plan. Variant data is the field most likely to be missing from an item-level schema anywhere in this market, and it is the one buyers complain about after the fact.

👉 [Test the eBay product endpoint on the free plan](https://www.scraperapi.com/?fp_ref=coupons)

## The search side, before you parse anything

You usually need discovery, not just enrichment of URLs you already have. ScraperAPI's async eBay search endpoint takes a `query`, or a `queries` array for batches, and posts the result to a webhook callback so your pipeline doesn't poll. Two parameters do the heavy lifting for international work: `country_code` and `tld`. The documentation states plainly that `country_code` influences **the language and the currency of the page**, which is how you get UK, German, Canadian and Australian pricing and shipping without rebuilding your parser per site.

Filters are passed in the request rather than applied afterwards: `condition` (with values like `used` or `open_box`, and multiple values allowed URL-encoded), plus a `show_only` set covering `returns_accepted`, `authorized_seller`, `completed_items`, `sold_items`, `sale_items`, `listed_as_lots`, `search_in_description`, `benefits_charity` and `authenticity_guarantee`. Filtering server-side is cheaper than pulling ten thousand rows and discarding most of them, which matters once you're paying per credit.

## How eBay burns through credits

One credit buys a standard page request, but the site and the parameters change the price. Amazon costs 5 credits, Google and Bing 25, LinkedIn 30, and **sites behind bot protection like Cloudflare, Datadome or PerimeterX add 10 credits per request when the bypass runs**. eBay defends its pages aggressively, so the honest way to size a plan is to run 20 or 30 of your real target URLs through the Domain Cost Estimator in the dashboard and multiply. A plan's credit count and the number of eBay pages you get out of it are not the same number.

You can also set a `max_cost` per scrape, which caps what a single request is allowed to consume. That's worth using on unfamiliar categories where you don't yet know whether a page will trigger a bypass.

👉 [Check plan allowances before you size your eBay volume](https://www.scraperapi.com/pricing?fp_ref=coupons)

## What building it yourself actually involves

Plenty of guides say self-hosting is "more work". Here is the specific work, taken from a public eBay scraper built on Scrapfly's unblocker:

- **The description is not on the page.** eBay renders item descriptions in an iframe (`iframe#desc_ifr`) that loads as an independent document. Your scraper has to capture that URL and fetch it separately, which doubles the request count for any job that needs description text.
- **Variants hide in JavaScript.** Per-variant price, stock and images sit in a `MSKU` variable inside a script tag, under `variationsMap`. Quantity available comes from a `NumberValidation` entry's `maxValue` inside `quantityInput.validations`. You're parsing page state, not markup, so a front-end change breaks you silently.
- **Pagination is arithmetic.** Total results come from `.srp-controls__count-heading>span`, page size from the `_ipg` URL parameter (default 60), page count from the division, and each further page from `_pgn`. Then you fan out concurrently.
- **eBay needs an anti-bot bypass at the transport layer.** That same codebase turns the unblocker on for every request with a comment stating eBay requires it. A separate Apify actor handles blocking with a Chrome TLS/HTTP fingerprint library and a proxy ladder that starts direct, falls back to datacenter, then residential, up to three attempts, and sticks to residential once one succeeds.

Then there's maintenance. The open-source Scrapy eBay scraper ships two spiders, user-agent rotation middleware, and three pipelines for validation, cleaning and price normalisation, plus CSV and JSON export across US, UK, DE, CA and AU sites. Its README reports roughly **100 listings per minute**, 30 detailed products per minute, over 95% field extraction and over 90% success against anti-bot measures. Those are the project's own figures, not an independent benchmark, and they come with the cost the README doesn't quantify: proxies, servers, and somebody to fix the parser when eBay changes a class name.

## The alternatives, side by side

| Option | How it works | Price visibility | Watch out for |
| --- | --- | --- | --- |
| Open-source Scrapy project | Two spiders, search and product; CSV/JSON export; US/UK/DE/CA/AU | Free code; you pay proxies and hosting | You own every breakage; benchmarks are self-reported |
| Apify "Ebay Scraper" | Accepts search, category or item URLs; `maxItems` cap; country proxy routing; parses JSON-LD plus buy box plus item specifics | No price on the actor page | Falls back direct → datacenter → residential, then sticks to residential |
| Apify "Ebay Product Scraper" | Takes a URL list, returns one record per URL; description as both plain text and HTML | No price on the actor page | Failed rows still return an object with `success: false` and an error string |
| Scrapfly sample code | Async Python, unblocker required, variant parsing included | No price on the sample | You still host, operate and monitor it |

None of those four published a price where I could verify it, so there's no cost comparison to draw. What is comparable is scope: ScraperAPI is the only one here with a documented eBay product endpoint and a documented async eBay search endpoint behind one API key, which is the difference between writing a pipeline and writing a parser plus a pipeline.

## Matching the plan to the job

**A few hundred SKUs, checked weekly.** Run the estimator on your URLs, then take the cheapest paid tier that covers the monthly total with room for a re-run after a failure. Concurrency above 5 rarely changes your wall-clock time at this size; a wider pipe just means more parallel failures.

**Tens of thousands of listings, daily.** You want overflow that doesn't stop the job. That's Scaling or Professional, where pay-as-you-go sits next to auto-upgrade and you set the monthly cap yourself. If your load is front-loaded into a launch window, early renewal on an annual plan pulls the next credit batch forward.

**Ad-hoc lookups.** Stay on the free plan. 1,000 credits covers a surprising amount of one-off enrichment, and the Domain Cost Estimator tells you when you're about to run out.

**Parsing control you can't give up.** If legal or architectural constraints mean data can't leave your infrastructure, none of the hosted options work and the Scrapy route is your starting point. Budget for the description iframe and the variant JSON before you budget for proxies.

## Before you pay

Run a real sample first, not a demo URL. Twenty to fifty of your own eBay links through the estimator tells you the credit cost per page far better than any tier comparison.

Know your failure mode. Credits running out pauses the job by default, so decide in advance whether you want auto-upgrade or a pay-as-you-go cap, and set that before a weekend run.

Confirm your target sites and currency behaviour. `country_code` and `tld` control language and currency on the search endpoint, so a US-configured job pointed at UK listings gives you numbers you'll have to throw away.

👉 [Open the pricing page and pick the tier your eBay volume actually needs](https://www.scraperapi.com/pricing?fp_ref=coupons)
