---
name: deal-check-ro
description: Check whether a product price at a Romanian online store is a good deal. Given a product link from eMAG, Altex, PC Garage, Flanco, CEL, evoMAG, amazon.de or another store, it finds the product code, looks up price history on istoric-preturi.info and pricy.ro, checks any extra sites the user names, and reports historical lows, a price history chart and current prices per store (cheapest first, or popular stores first). Use when the user shares a Romanian retailer product link or asks "is this a good price", "e bun prețul?", "istoric preț", "cel mai mic preț", for price history, or where to buy something cheapest in Romania.
license: MIT
compatibility: Needs network access. agent-browser is optional and adds price charts and some browser-only pages.
allowed-tools: Bash(curl:*) Bash(python3:*) Bash(agent-browser:*) Bash(cat:*) Bash(ls:*) WebFetch WebSearch
metadata:
  author: razgraf
  version: "1.0.0"
---

# Deal check (Romania)

Given a product link, report whether today's price is a good deal: current prices per store, historical lows, and a price chart.

## Personal use and data sources

This skill is for **personal, non-commercial use**: one person checking a product they're thinking of buying. Don't use it for bulk scraping, price monitoring at scale, reselling data, or building datasets.

All data comes from third-party websites. The user is responsible for respecting each site's terms of service and its policies on automated and AI access. Sites can restrict, change or stop providing this data at any time, and the skill will then return less. Prices are indicative and may be out of date. The skill isn't affiliated with any site it reads.

Follow the access rules in [references/sources.md](references/sources.md) on every run: respect robots.txt, never get around a challenge page or captcha, and never fetch amazon.de.

## Inputs

- A product URL, or a product name or model code. Strip tracking parameters (`?ref=…`, `utm_…`, `#…`) from URLs before using them.
- Sort mode (optional): **lowest price first** by default; "popular first" / "magazine populare" switches to the store ranking in [references/report.md](references/report.md).
- Extra sites (optional): stores or price sites the user wants checked too ("also check vexio.ro", "verifică și pe compari"). See "Extra sites" in step 3.

## Workflow

### 1. Prepare the browser

Run the setup in [references/browser.md](references/browser.md) once, before anything else: check that agent-browser is installed and open the isolated `deal-check-ro` session (it borrows an installed Chrome, Chromium, Brave or Edge if agent-browser's own browser is missing). Note whether a browser is available; the steps below depend on it. Other browser tools the agent already has can stand in for agent-browser.

If agent-browser isn't installed, tell the user this once, in their language, then carry on without waiting for an answer:

> agent-browser isn't installed, so this check skips the full price chart and pages that need a browser. Install it for full results and faster browser steps: `npm i -g agent-browser` (https://agent-browser.dev).

### 2. Identify the product

Skip this step if the user gave a name or code instead of a link.

1. Detect the retailer from the host and extract its ID ([references/retailers.md](references/retailers.md)).
2. Read the page's JSON-LD `Product`: `name`, `brand`, `mpn`, `gtin`, current price, availability.
   - Hosts that retailers.md marks ❌ (Altex, PC Garage, Flanco): don't fetch them, not even in the browser. Take the name and code from the URL and let istoric-preturi's lookup (step 3) fill in the rest.
   - Other hosts: plain fetch with the `curl` command in retailers.md. If the response is blocked, try the browser once.
3. If neither works (no browser, a challenge page, a connection error), take the name from the URL slug. Ask the user for the model code only if the slug is ambiguous.
4. amazon.de: never fetch. Take the ASIN and the name from the slug. For short links (`amzn.eu/d/…`) or URLs without a slug, ask the user for the product name.

Match key: **mpn** first, then **gtin**, then brand + model name (+ storage/size/color). Write the key down; every later match is checked against it.

### 3. Collect offers and history

Run the independent lookups in parallel when the agent supports it. Details, URL patterns and disallowed paths are in [references/sources.md](references/sources.md).

| Source | No browser | With browser |
|---|---|---|
| istoric-preturi.info | POST the product URL to get the product page; read offers table + 30/90-day changes. At most 2 pages per run (sources.md) | same page in the browser, + per-store history chart |
| pricy.ro | one domain-restricted web search by name; offers | + min/avg history chart |
| compari.ro | skipped, blocks automated access | skipped, unless the user names it as an extra site |
| eMAG, CEL | site search by mpn, then by name if nothing matches; JSON-LD price | same |
| evoMAG, other stores | domain-restricted web search; plain fetch of the product page | same, browser if blocked |
| Altex, PC Garage, Flanco | not fetched; prices from istoric-preturi | same, unless the user names one as an extra site |
| amazon.de | search link only | search link only |

Skip the store search for the store the user's link came from; you already have that page.

If istoric-preturi is blocked, carry on with the other sources. report.md says how to report it.

**Web searches**: when the search tool has a domain filter (Claude's WebSearch: `allowed_domains`), use it; `site:` written in the query is ignored by some tools. Search by product name (brand, model, key variant) plus mpn.

**Extra sites**: check each site the user named, up to 5 per run.
1. Skip amazon.de and anything under "Not used" in sources.md, and say why in one line.
2. Read the site's `/robots.txt` once and don't request paths it disallows.
3. Find the product page with a domain-restricted web search by name and mpn. Don't use the site's own search unless robots.txt allows it.
4. Read the page's JSON-LD: plain fetch first, one browser try if blocked. For sites marked as blocking (Altex, PC Garage, Flanco, compari.ro), go straight to the one browser try.
5. Validate the offer as in step 4 and add it to the table. If the page draws a Chart.js price chart and istoric-preturi gave none, read it with the chart script in browser.md.

If a named site can't be read, the report says so in one line (see report.md).

**Outdated links**: if the user's page can't be read and istoric-preturi doesn't recognize the link, search the store's domain for the product name. If a current page turns up, use it and tell the user which URL you used.

### 4. Validate every match

Keep an offer only if:
- its model code matches the key after normalizing both (lowercase, drop `/`, `-`, `.`, `_` and spaces, so `MG6J4ZD/A` = `mg6j4zda`; store-prefixed codes like `J4ZDA_MG6J4ZDA` match on the part after `_`), or its title names the same model **and** the same variant (storage, size, color when the price differs by color);
- the condition is new. **Drop** refurbished, resigilat, "deschis", used, "second hand", and international/"Indian spec" versions unless the user's own link is one;
- it's the product itself. A free-game or voucher promo on the same product still counts. Drop cases, accessories, and bundles that add another physical product.

When unsure, leave the offer out.

### 5. Compute and report

Follow [references/report.md](references/report.md): normalize store names and dedupe, flag and order stale offers, compute the lows, pick a verdict, sort, draw the price history chart and end with the personal-use footer. Keep the report to that template: no sources list and no closing summary.

## Rules

- Every price in the report comes from a page read in this run and has a check date. Never estimate or fill gaps beyond the carry-forward rule in report.md.
- Don't fetch robots-disallowed paths, `/out/`/`Jump.php` redirect links, or anything on amazon.de.
- A blocked plain fetch may be tried **once** in the browser. A challenge page, captcha or connection error **in the browser** ends that source for this run: don't wait it out, click through, solve it, or retry with different headers or tools.
- Keep requests minimal: only the pages this product needs, one at a time per site.
- No buying, carting, logging in or price alerts. The skill only reads.
- Use only the `deal-check-ro` session: never `--profile`, never `--cdp` into the user's own browser. Close it when done (`agent-browser --session deal-check-ro close`).
