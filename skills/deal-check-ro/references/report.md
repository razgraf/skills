# Report format

Write in the user's language: Romanian if they wrote in Romanian, otherwise English. Number format follows the language: `4.399,99 lei` (RO) or `4,399.99 RON` (EN).

## Computations

Take only offers that passed the match check (same model code and variant, new condition, no accessories, no refurbished/resigilat/used).

### Current offers

- **Store names**: sources name stores differently (`PC Garage` on istoric-preturi, `pcgarage.ro` on compari and pricy). Compare names lowercased with `www.`, the TLD, spaces, hyphens and dots removed (`pcgarage.ro` → `pcgarage` = `PC Garage`). Display the brand-style name.
- **Dedupe**: one row per store. If two sources list the same store, keep the most recently checked price.
- **Stale**: last check more than 14 days ago. Keep the row, mark it `⚠️ stale`, and leave it out of best now. istoric-preturi's own "A trecut … mult timp" notes don't decide staleness; the check date does.
- **Too old**: last check more than 90 days ago. Leave the row out of the table and write in Sources how many offers were dropped as too old.
- **Best now**: the cheapest non-stale offer.

### History

istoric-preturi's per-store chart is the main history. Its points mark price changes, not daily prices, so turn each store's points into a daily series like this:

1. A point's price holds from its date until the store's next point.
2. After a store's last point, the price holds until the store's last check date from the offers table, or 45 days after the last point if the store isn't in the table. After that the store has no price.
3. The **history series** is the lowest price across all stores for each day.

Lows come from that series: **all-time low**, **90-day low** and **30-day low** (windows counted back from today), each with its date and store.

compari's `preţ minim` and pricy's `Cel mai mic preț` are the minimum across every offer they list, including other colors, international versions and offers that step 4 would reject. Use them for lows only when there's no istoric-preturi chart, and then end the verdict line with "(based on <source>'s minimum across all listings)". When you have both, use the comparator series only as context (e.g. in the chart caption).

### Without a chart (partial history)

When no chart was read, the only history is istoric-preturi's 30-day and 90-day columns. Each gives a past price per store ("de la X RON"). A "0%" / "a rămas la fel" cell means the price didn't change, and `?` means no data.

- These are samples from about 30 and 90 days ago, not window minimums. Date them "~30 days ago" and "~90 days ago". Don't invent exact dates.
- The **reference low** is the lowest of those samples and today's non-stale prices. Call it "lowest price seen" (never "90-day low"), and title the table column "vs. lowest seen".
- Skip the 🟢 verdict rule; the best possible verdict is 🟡. End the verdict line with "(partial history, no chart)".
- Leave out the ASCII chart and the all-time line under Historical lows.

With a chart, the reference low is the all-time low and the column is "vs. all-time low".

### Verdict

Compare **best now** against the lows:

| Condition | Verdict (EN / RO) |
|---|---|
| best now ≤ all-time low × 1.02 | 🟢 Great deal: at the lowest recorded price / Ofertă foarte bună: la cel mai mic preț înregistrat |
| best now ≤ 90-day low × 1.05 | 🟡 Fair price: close to recent lows / Preț corect: aproape de minimele recente |
| otherwise | 🔴 Not a deal: N% above the 90-day low / Nu e o ofertă: cu N% peste minimul din 90 de zile |

In partial-history mode, read "90-day low" as the reference low and word it "N% above the lowest price seen" / "cu N% peste cel mai mic preț văzut".

Add one line of context when the data supports it:
- **Inflated discount**: the store shows a strikethrough "old" price, but its own history shows it was below today's price in the last 90 days.
- **Trend**: the price has fallen 3+ months in a row, so waiting may pay off. Say so as an observation, not advice.
- **Seasonal**: Black Friday (November) and post-launch drops for phones are common. Mention them only if the history shows such a pattern for this product.

## Sorting

Fresh rows come first, then stale rows, each group sorted by the chosen mode. amazon.de is always last.

- **Lowest price first** (default).
- **Popular first** ("popular first" / "magazine populare"): this fixed ranking, then the remaining stores by price:

  1. eMAG 2. eMAG marketplace 3. Altex 4. Flanco 5. PC Garage 6. CEL 7. evoMAG 8. Vexio 9. Carrefour 10. Dedeman 11. Elefant 12. F64 13. iStyle 14. Ozone 15. ITGalaxy

## ASCII chart

Draw it only from chart data (the daily history series above), and only when it fills at least 4 buckets. Use monthly buckets with the lowest price in each month, or weekly buckets if the history spans less than 60 days. Map each bucket to one of `▁▂▃▄▅▆▇█`:

```
level = round((value - min) / (max - min) * 7)   # 0 → ▁, 7 → █; if max == min use ▄
```

Months with no data show `·`. Print it in a fenced code block so the characters line up:

```
Lowest price per month · <first month> → <last month>
▅▅▄▃▃▄▂▁▂▃▅▇█
min <price> (<month>) · max <price> (<month>)
```

## Template

```markdown
## <Product name> · <mpn>
<verdict line>
<one line of context, if any>

### Current prices (<sort mode>)
| Store | Price | vs. <reference low> | Checked | Link |
|---|---|---|---|---|
| <store> | <price> | +<n>% | <n> days ago | [open](<url>) |
| <store> ⚠️ stale | <price> | +<n>% | <n> days ago | [open](<url>) |
| amazon.de | check manually (EUR) | n/a | n/a | [search](https://www.amazon.de/s?k=<mpn>) |

### Historical lows
- All-time: <price> · <store or source> · <date>
- 90 days: <price> · <store> · <date> · 30 days: <price> · <store> · <date>

### Price history
<ASCII chart block>

### Sources
- ✅ <source> (<what it gave>) · <url>
- ⚠️ <source>: no matching product found
- ⏭️ <source>: skipped, needs a browser
- ❌ <source>: challenge page or connection error

<footer>
```

The placeholders are only placeholders. Every value must come from this run's data.

Leave out any section with no data instead of showing an empty table. Always keep Sources and the footer.

**Links.** Point to the retailer product page when you know its URL, otherwise to the aggregator product page. If you couldn't read the retailer page yourself, the price comes from the aggregator; name it in the Checked cell ("2 days ago, via istoric-preturi"). Never link an aggregator's `/out/` or `Jump.php` redirect.

## Install hint

Add this under Sources when browser-only steps were skipped (no agent-browser, or no browser it could launch):

> Price charts need a browser. To include them, install agent-browser: `npm i -g agent-browser`. It uses Chrome, Brave or Edge if you have one, otherwise run `agent-browser install` once to download Chromium.

## Footer

Always end the report with one line:

- EN: `_Personal use only. Prices come from third-party sites, may be out of date, and each site's terms apply. Check the store before buying._`
- RO: `_Doar pentru uz personal. Prețurile provin de pe site-uri terțe, pot fi neactualizate și se supun termenilor fiecărui site. Verifică pe site-ul magazinului înainte de cumpărare._`
