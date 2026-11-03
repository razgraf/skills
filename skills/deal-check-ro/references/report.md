# Report format

Write in the user's language: Romanian if they wrote in Romanian, otherwise English. Number format follows the language: `4.399,99 lei` (RO) or `4,399.99 RON` (EN).

## Computations

Take only offers that passed the match check (same model code and variant, new condition, no accessories, no refurbished/resigilat/used).

### Current offers

- **Store names**: sources name stores differently (`PC Garage` on istoric-preturi, `pcgarage.ro` on compari and pricy). Compare names lowercased with `www.`, the TLD, spaces, hyphens and dots removed (`pcgarage.ro` → `pcgarage` = `PC Garage`). Display the brand-style name.
- **Dedupe**: one row per store. If two sources list the same store, keep the most recently checked price.
- **Stale**: last check more than 14 days ago. Keep the row, add `⚠️` after its check date, and leave it out of best now. istoric-preturi's own "A trecut … mult timp" notes don't decide staleness; the check date does.
- **Too old**: last check more than 90 days ago. Leave the row out.
- **Best now**: the cheapest non-stale offer.

### History

istoric-preturi's per-store chart is the main history. Its points mark price changes, not daily prices, so turn each store's points into a daily series like this:

1. A point's price holds from its date until the store's next point.
2. After a store's last point, the price holds until the store's last check date from the offers table, or 45 days after the last point if the store isn't in the table. After that the store has no price.
3. The **history series** is the lowest price across all stores for each day.

Lows come from that series: **all-time low**, **90-day low** and **30-day low** (windows counted back from today), each with its date and store.

compari's `preţ minim` and pricy's `Cel mai mic preț` are the minimum across every offer they list, including other colors, international versions and offers that step 4 would reject. Use them for lows only when there's no istoric-preturi chart, and then add the comparator note under the chart (see "Note line"). When you have both, ignore the comparator series.

### Without a chart (partial history)

When no chart was read, the only history is istoric-preturi's 30-day and 90-day columns. Each gives a past price per store ("de la X RON"). A "0%" / "a rămas la fel" cell means the price didn't change, and `?` means no data.

- Use only non-stale stores, so the samples line up in time.
- These are samples from about 30 and 90 days ago, not window minimums. Label them "~30 days ago" and "~90 days ago". Don't invent exact dates.
- The **reference low** is the lowest of those samples and today's non-stale prices.
- The best possible verdict is 🟡.

With a chart, the reference low is the all-time low.

### No history

When there's no chart and no 30/90-day samples (istoric-preturi was blocked or doesn't track the product, and pricy had no chart):

- No verdict dot. The verdict line is `No price history. Cheapest of <n> stores checked: <price> at <store>.` / `Fără istoric de preț. Cel mai ieftin din <n> magazine verificate: <price> la <store>.`
- The table's "vs. lowest" column becomes "vs. cheapest" and compares with best now.
- No Price history section and no context line.

### Verdict

Compare **best now** against the lows. The verdict line is the dot, the label, then one sentence with the best price, the store and how it compares:

| Condition | Label (EN / RO) | Sentence |
|---|---|---|
| best now ≤ all-time low × 1.02 | 🟢 Good deal / Ofertă bună | `<price> at <store>, the lowest price on record.` |
| best now ≤ 90-day low × 1.05 | 🟡 Fair price / Preț corect | `<price> at <store>, <n>% above the 90-day low.` or `…, the lowest price seen.` when it matches the reference low |
| otherwise | 🔴 Not a deal / Nu e o ofertă | `<price> at <store>, <n>% above the 90-day low.` |

In partial-history mode, read "90-day low" as the reference low and say "the lowest price seen" / "cel mai mic preț văzut".

Add one line of context only when the data shows it:
- **Inflated discount**: the store shows a strikethrough "old" price, but its own history shows it was below today's price in the last 90 days.
- **Trend**: the price has fallen 3+ months in a row. State the drop, not advice.
- **Seasonal**: Black Friday (November) and post-launch drops for phones are common. Mention them only if this product's history shows such a drop.

## Sorting

Fresh rows come first, then stale rows, each group sorted by the chosen mode. amazon.de is always last.

- **Lowest price first** (default).
- **Popular first** ("popular first" / "magazine populare"): this fixed ranking, then the remaining stores by price:

  1. eMAG 2. eMAG marketplace 3. Altex 4. Flanco 5. PC Garage 6. CEL 7. evoMAG 8. Vexio 9. Carrefour 10. Dedeman 11. Elefant 12. F64 13. iStyle 14. Ozone 15. ITGalaxy

## Price history chart

Draw one whenever there's any history. Print it in a fenced code block so the bars line up. Each row is a label, the price right-aligned, and a bar of `█`:

```
bar = 4 + round((value - min) / (max - min) * 16)   # if max == min, every bar is 12
```

**From chart data** (the daily history series above): one row per month with that month's lowest price. If the history spans more than 12 months, use one row per quarter (`Q3 2025`) instead; under 60 days, one row per week (`20 Oct`). A month with no data gets `no data` and no bar.

```
Lowest price per month
Sep 2025  7,929.99 RON  ████████████████████
Oct 2025  7,799.00 RON  █████████████████
…
Nov 2026  7,099.00 RON  ████
```

Under the block, the **lows line**. Name each distinct low once:
- All three lows differ: `All-time <price> (<store>, <date>) · 90 days <price> (<store>, <date>) · 30 days <price> (<store>, <date>)`.
- Drop the 30-day low when it equals the 90-day low, and the 90-day low when it equals the all-time low.
- All equal: `Lowest on record: <price> (<store>, <date>)`. Add `, today's price` when best now matches it.

**From samples** (partial history): one row per sample, the lowest price across stores at each point. No lows line; the bars show them.

```
Lowest price across stores
~90 days ago  7,905.00 RON  ████████████████████
~30 days ago  7,599.00 RON  ██████████████
today         7,099.00 RON  ████
```

**Note line.** At most one short line after the chart, and only for the first case that applies:
1. Partial history: why the full chart is missing. "Two past samples only. The full chart needs agent-browser." or "Two past samples only. istoric-preturi showed a browser check."
2. The lows come from a comparator minimum: "Lows are <source>'s minimum across all its listings, other colors included."
3. The chart covers only some of the stores in the table: "History covers <store> only." or "History covers <n> of <m> stores."

## Template

```markdown
## <Product name> · <mpn>
<verdict line>
<one line of context, if any>

| Store | Price | vs. lowest | Checked | |
|---|---|---|---|---|
| <store> | <price> | +<n>% | today | [open](<url>) |
| <store> | <price> | +<n>% | <n> days ago ⚠️ | [open](<url>) |
| amazon.de | check yourself (EUR) | | | [search](https://www.amazon.de/s?k=<mpn>) |

### Price history
<chart block>
<lows line, if chart data>
<note line, if any>

<footer>
```

The placeholders are only placeholders. Every value must come from this run's data.

If istoric-preturi was blocked (challenge page, captcha or connection error), add one line under the table: "istoric-preturi blocked access this time, so Altex, PC Garage, Flanco and smaller stores are missing." Add "and there's no price history" when pricy gave none either.

If an extra site the user named couldn't be read, add one line under the table per site: "Couldn't check <site>: <reason in a few words>." Don't list other skipped sources.

"vs. lowest" compares with the all-time low, or with the reference low in partial-history mode, or with best now in no-history mode. Leave out the Price history section when there's no history at all.

Nothing goes after the template except the footer: no sources list, no summary, no "bottom line" or buying advice. If you used a different URL than the user's (outdated link), say so in one line under the title.

**Links.** Point to the retailer product page when you know its URL, otherwise to the aggregator product page. Never link an aggregator's `/out/` or `Jump.php` redirect.

## Footer

Always end the report with one line:

- EN: `_Personal use only. Prices come from third-party sites, may be out of date, and each site's terms apply. Check the store before buying._`
- RO: `_Doar pentru uz personal. Prețurile provin de pe site-uri terțe, pot fi neactualizate și se supun termenilor fiecărui site. Verifică pe site-ul magazinului înainte de cumpărare._`
