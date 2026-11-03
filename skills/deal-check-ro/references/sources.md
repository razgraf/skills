# Price history and comparison sources

Verified November 2026. Sites change. If a pattern below stops working, skip that source rather than guessing a new endpoint.

## Access rules (apply to every source)

- **robots.txt**: never request a path the site disallows for `User-agent: *` or for AI agents. The disallowed paths that matter are listed per source below. If you need a path that isn't covered here, read the site's `/robots.txt` first.
- **Blocked pages**: a plain fetch that's blocked may be tried once in the browser. A challenge page ("Just a moment…", captcha, "verify you are human") or a connection error in the browser means the source is unavailable for this run. Don't wait it out, solve it, change headers or retry.
- **Outbound links** (`/out/`, `/link/`, `Jump.php`, "spre magazin"): show them to the user as links, but never fetch them.
- **Pace**: at most one request per source per few seconds, and only the pages this product needs.
- **Web searches**: use the search tool's domain filter when it has one (Claude's WebSearch: `allowed_domains: ["<host>"]`). Some tools ignore `site:` written in the query and return unrelated pages. Accept only results on the expected host and URL pattern.

## 1. istoric-preturi.info (first choice)

Covers 500+ stores, gives per-store history and has a lookup that accepts a retailer URL. It's the main history source and the only source for Altex, PC Garage, Flanco and most small stores.

**Keep requests low.** Its browser check has started appearing after a few requests in a row, so per run send at most the lookup plus 2 pages, at least 5 seconds apart:
1. The lookup `POST` (plain curl, don't follow the redirect).
2. The product page. With a browser, open it there only and read the offers, the JSON-LD and the chart from that one page. Without one, a plain fetch.
3. At most one more page: the multi-store page from "Similare", read the same way.

Skip variant pages and anything else. If any step is blocked, a plain fetch gets one browser try; a challenge in the browser ends istoric-preturi for the run. Carry on with the other sources (see "No history" in `report.md`).

| What | How |
|---|---|
| Lookup by retailer URL | `POST https://www.istoric-preturi.info/` with form field `link=<retailer product URL without query string>`. It answers `302` with `Location: /pd/<id>/<code>/<slug>`. Plain curl works: `curl -s -o /dev/null -w '%{redirect_url}' -A '<UA>' -X POST --data-urlencode 'link=<url>' https://www.istoric-preturi.info/` |
| Not tracked | The lookup redirects to `/search/...` instead, which is **disallowed**. Don't follow it. Do one domain-restricted web search with the product name and mpn, then move on |
| Product page | `https://www.istoric-preturi.info/pd/<id>/<code>/<slug>`, plain fetch ✅ |
| Product info | JSON-LD `Product` (`name`, `brand`, `mpn`, `gtin12`; description lists the stores). "Cod produs" in the page text |
| Current offers | "De vanzare pe" table. Per store: price, last check ("acum N zile/luni"), 30-day change and 90-day change, each with the previous price ("de la X RON") |
| Single-store pages | The lookup can land on a page tracking only the store you came from (one row in "De vanzare pe", a code like `J4ZDA_MG6J4ZDA`). Look in the "Similare" block for an entry with the same mpn or the same title, and use that page too. It's usually the multi-store one. Other stores can have their own entries there as well (e.g. `EVOMAG-4247198`); match them by title |
| Variants | "Variante" block lists sibling variants (storage/color) with their own prices |
| Staleness text | "A trecut cam mult timp de la ultima verificare" (appears after a few days) and "A trecut destul de mult timp…" (months). Informational only; staleness is decided by the check date (see `report.md`) |
| Other cell values | `?` in the 30/90-day columns means "Pretul nu a mai putut fi verificat" (no data, don't use it). "SITE NERECOMANDAT" next to a store is istoric-preturi's own warning; keep the row and write `(flagged by istoric-preturi)` after the store name |
| History chart | Chart.js on `#pricesChart`, one dataset per store. Labels like `23 Oct 2025` use Romanian month abbreviations, values are strings, `null` where nothing changed. The data is obfuscated in the HTML, so it needs a browser (see `browser.md`). Points mark price changes, not daily prices |
| Disallowed | `/search`, `/brands_search`, `/out/`, `/link/` |

No browser: use the 30-day and 90-day "de la X RON" values as history samples (see "Without a chart" in `report.md`). A "0%" cell has no "de la" value; the past price equals today's.

## 2. compari.ro (largest RO comparator, rarely reachable)

Skipped by default: it showed a Cloudflare challenge on every page, in plain fetches and in headless agent-browser (November 2026). Check it only when the user names it as an extra site, with one browser try. If the title is a challenge, move on.

| What | How |
|---|---|
| Find product | Domain-restricted web search with the product name. The on-site search (`/CategorySearch.php?st=`) is **disallowed** |
| Product page | `https://www.compari.ro/<category>-c<id>/<brand>/<slug>-p<productId>/`. One page often covers **all colors** of a model |
| Current offers | Listed under "Magazine şi preţuri": store domain, offer title (check variant, color and code in it), price, delivery cost, stock |
| History chart | Chart.js canvas `#chartnormal`, two datasets: `preţ minim` (lowest across **all** listed offers, including other colors, international versions and unvalidated sellers) and `preţ mediu` (average). Points are `{t: <epoch ms>, y: <RON>}`. See `report.md` for how much to trust it |
| Disallowed | `/Ajax*`, `/Jump.php`, `/outgoing/`, `/*?st=`, `/*/fn:*`, `/*/f:pret=` |

## 3. pricy.ro (good history, hard to locate)

| What | How |
|---|---|
| Find product | One domain-restricted web search with the **product name** (brand, model, storage, color). Searching by bare mpn mostly returns nothing. Accept only `/ProductUrlId/` results; ignore `/productsv2?q=` results (disallowed). Skip pricy if nothing matches |
| Product page | `https://www.pricy.ro/ProductUrlId/<slug>-<32 hex>`, plain fetch ✅ |
| Current offers | JSON-LD offers (seller, price, `priceValidUntil`) and the "Toate ofertele" table. The header shows "CEL MAI MIC PREȚ" and "Actualizat acum N zile" |
| History chart | Lazy-loaded. Click the "Istoric preț" tab (`a[href="#price-history"]`), then read the Chart.js chart on `#priceHistoryCanvas`: datasets `Cel mai mic preț` (lowest across all listed offers, same caveat as compari) and `Preț mediu`; labels are `Date` objects. Range buttons: "3 luni", "6 luni", "1 an" (default) and "Tot". Browser only |
| Disallowed | `/api/`, `/productsv2?`, `/Product/Json` (the chart's own data call, so don't fetch it yourself) |

## 4. Retailers directly

See `retailers.md`. Only two retailers have searches robots.txt allows **and** that a plain fetch can read:

- eMAG: `https://www.emag.ro/search/<q>`
- CEL: `https://www.cel.ro/cauta/<q>/`

Search first with the full, URL-encoded mpn (`MG6J4ZD%2FA`); dropping the slash returns unrelated products on eMAG. If no result's slug contains the model (CEL often misses codes with dashes, like `PRIME-RTX5070TI-O16G`), search once more with brand + model name. Results include filler products, so open only links whose slug contains the model, then validate the JSON-LD `mpn`.

For any other store, find its product page with a domain-restricted web search, then read the JSON-LD from that page (plain fetch, or one browser try if blocked).

## Not used

| Source | Why |
|---|---|
| amazon.de | robots.txt disallows AI agents for the whole site. Link only, see `retailers.md` |
| Google Shopping | ToS forbids automated queries |
| price.ro | Cloudflare-blocked, not evaluated |
| shopmania.ro | search relevance too poor (mostly accessories) |
| cumpar.ro | domain no longer a price site |
| Browser extensions (xPrice, Pretzi, Istoric Pret) | no public API |
