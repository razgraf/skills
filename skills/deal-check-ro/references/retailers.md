# Retailers

How to identify a product from a retailer URL and read its current price.

Every retailer below publishes a schema.org `Product` in `<script type="application/ld+json">`. That is the primary source for name, brand, codes and price. Fall back to `og:title` and visible text only when JSON-LD is missing.

Use this `curl` invocation for every plain fetch:

```bash
curl -sL --compressed -m 20 \
  -A 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36' \
  -H 'Accept-Language: ro-RO,ro;q=0.9,en;q=0.8' \
  '<url>'
```

A plain fetch **worked** when it returns a page with a parseable Product JSON-LD (the one-liner below prints a line). Anything else (an error status, a connection reset, a challenge page, a page without Product JSON-LD) counts as **blocked**. Don't look for challenge keywords: ordinary pages mention "captcha" in their config too. A blocked page may be tried once in the browser (see `browser.md`). Never retry with altered headers.

## Per-retailer table

| Retailer | Host | Product URL → ID | Plain fetch | Search allowed by robots.txt |
|---|---|---|---|---|
| eMAG | `emag.ro` | `/pd/([A-Z0-9]+)` → PNK (also JSON-LD `sku`) | ✅ | ✅ `https://www.emag.ro/search/<q>` |
| Altex | `altex.ro` | `/cpd/([A-Z0-9]+)` → SKU (= mpn without `/`, e.g. `MG6J4ZDA`) | ❌ HTTP/2 error, also in headless browsers | ❌ `/cauta/` disallowed |
| PC Garage | `pcgarage.ro` | no ID in URL (`/<category>/<brand>/<slug>/`); JSON-LD `sku` (numeric), `mpn` | ❌ Cloudflare, also in headless browsers | not used |
| Flanco | `flanco.ro` | `/<slug>.html`; ID from JSON-LD | ❌ Cloudflare, also in headless browsers | not used |
| CEL | `cel.ro` | `/<slug>-p<code>-l/` | ✅ | ✅ `https://www.cel.ro/cauta/<q>/` |
| evoMAG | `evomag.ro` | ID from JSON-LD | ✅ | ❌ `?sn.q=` disallowed |
| Vexio, Carrefour, Elefant, others | varies | read JSON-LD | try plain, else browser once | not used |
| amazon.de | `amazon.de` | `/dp/([A-Z0-9]{10})` or `/gp/product/([A-Z0-9]{10})` → ASIN | **never fetch** | **never fetch** |

"Not used" means the skill doesn't use that store's own search. Find its product page with a domain-restricted web search instead, then open only that page.

For ❌ stores, skip the plain fetch; it can't work and Altex can hang until the timeout. With a browser, try the product page once. Either way, their prices usually come from istoric-preturi. Smaller stores that only appear in aggregator listings (ITArena, Flax, Ozone…) aren't fetched directly; the aggregator's price and check date are enough.

When you link a store page you couldn't read yourself, the price shown comes from the aggregator. Say so with the aggregator as the source.

## JSON-LD fields to read

From the object whose `@type` is (or includes) `Product`. It may sit inside `@graph` or an array.

| Field | Use |
|---|---|
| `name` | display name |
| `brand.name` | matching |
| `mpn` | **primary match key** (e.g. `MG6J4ZD/A`) |
| `gtin`, `gtin8`, `gtin12`, `gtin13` | secondary match key (EAN/UPC). eMAG has none |
| `sku` | store-internal ID, not portable across stores |
| `offers.price` / `offers.lowPrice` | current price (RON) |
| `offers.availability` | `InStock` / `OutOfStock` / `PreOrder` |
| `offers.itemCondition` | drop `UsedCondition`, `RefurbishedCondition`, `DamagedCondition`. **Missing counts as new** (eMAG omits it) |

**Comparing codes.** Stores format the same mpn differently: `MG6J4ZD/A` (eMAG, PC Garage), `MG6J4ZDA` (Altex `sku`), `mg6j4zda` (CEL). Normalize before comparing: lowercase, then remove `/`, `-`, `.`, `_` and spaces. istoric-preturi sometimes prefixes a store code (`J4ZDA_MG6J4ZDA`) or uses a store ID instead (`EVOMAG-4247198`). Compare the part after `_`, or fall back to title matching.

**eMAG marketplace.** The page's inline script has `EM.multiple_offers_count` and `EM.multiple_min_price`, the cheapest other seller on eMAG. That minimum may be a resigilat or used unit, so check the page's offer list (`EM.multiple_offers`) for that seller's offer first:
- Confirmed new and cheaper than eMAG's own price: show it as its own row, "eMAG marketplace (<seller>, check seller rating)". It may count as best now.
- Condition unconfirmed: leave it out of the table and best now. Mention it in Sources as "eMAG marketplace: N other offers from X RON, condition not verified".
- Not cheaper than eMAG's own price: leave it out.

Ignore `EM.used_offers`.

Some stores (CEL) wrap a JSON-LD block in an HTML comment. It fails to parse, which is fine; use the block that parses.

## Extracting with a one-liner

Works with any Python 3:

```bash
curl ... '<url>' | python3 -c '
import sys, re, json
html = sys.stdin.read()
def products(data):
    items = data if isinstance(data, list) else data.get("@graph", [data])
    for it in items:
        if not isinstance(it, dict): continue
        t = it.get("@type")
        if t == "Product" or (isinstance(t, list) and "Product" in t): yield it
for block in re.findall(r"<script[^>]*application/ld\+json[^>]*>(.*?)</script>", html, re.S):
    try: data = json.loads(block)
    except Exception: continue
    for it in products(data):
        o = it.get("offers") or {}
        o = o[0] if isinstance(o, list) else o
        brand = it.get("brand")
        out = {k: it.get(k) for k in ("name", "mpn", "gtin", "gtin8", "gtin12", "gtin13", "sku")}
        out.update({"brand": brand.get("name") if isinstance(brand, dict) else brand, "price": o.get("price") or o.get("lowPrice"), "availability": o.get("availability"), "condition": o.get("itemCondition")})
        print(json.dumps(out, ensure_ascii=False))
'
```

If `python3` is missing, read the JSON-LD block with `grep`/`sed` or use WebFetch on the URL and ask for the Product JSON-LD fields.

## amazon.de (link only)

amazon.de's robots.txt disallows ClaudeBot, Claude-User, GPTBot, Perplexity-User and other AI agents for the whole site. Never fetch amazon.de, not even in a browser, and that includes resolving `amzn.eu` / `amzn.to` short links.

- **amazon.de URL as input**: extract the ASIN and derive the name from the URL slug (`/Apple-iPhone-17-256-GB/dp/B0…` → "Apple iPhone 17 256 GB"). For a short link or a URL without a slug, ask the user for the product name or model code.
- **In every report**: add a search link `https://www.amazon.de/s?k=<url-encoded mpn or name>` labeled "check manually, prices in EUR, shipping to RO may apply".
