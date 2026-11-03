# Browser fallback (agent-browser)

Use a browser only when a plain fetch is blocked (one try per page), or to read a history chart the page draws itself.

Other browser tools (an in-app browser or Chrome extension the agent already has) are fine too. The scripts below are plain JavaScript and run in any of them.

## Setup (once per run)

**1. Is agent-browser installed?**

```bash
agent-browser --version
```

If not, show the install suggestion from SKILL.md step 1, skip the browser-only steps and finish the report with the data you have.

**2. Start the session.** Every command in this file uses `--session deal-check-ro`. A session without `--profile` starts from an empty, throwaway profile, so the user's cookies, logins and history are never touched. Never pass `--profile`, and never use `--cdp` to attach to the user's own running browser.

```bash
agent-browser --session deal-check-ro open about:blank
```

**3. If that open fails** (usually "Executable doesn't exist" or "Chromium not found": agent-browser's own browser isn't downloaded), close the half-started session first. A failed `open` leaves the session's daemon running, and a running daemon ignores `--executable-path`:

```bash
agent-browser --session deal-check-ro close
```

Then use a browser that's already installed. This lists the ones that exist; any of them works:

```bash
ls -1 \
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  "/Applications/Chromium.app/Contents/MacOS/Chromium" \
  "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser" \
  "/Applications/Microsoft Edge.app/Contents/MacOS/Microsoft Edge" \
  /usr/bin/google-chrome-stable /usr/bin/google-chrome /usr/bin/chromium \
  /usr/bin/chromium-browser /snap/bin/chromium /usr/bin/brave-browser /usr/bin/microsoft-edge \
  2>/dev/null
```

On Windows, check `C:\Program Files\Google\Chrome\Application\chrome.exe` and `C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe`.

Then open the session with that browser. `--executable-path` only matters on the `open` that launches the session; later commands reuse it:

```bash
agent-browser --session deal-check-ro --executable-path "<path>" open about:blank
```

If this fails too, run `close` again and continue without the browser.

The skill still gets a fresh, empty profile; it only borrows the browser program. Chromium's process sandbox stays on. Don't add `--no-sandbox`.

If no browser is found, tell the user they can run `agent-browser install` once (it downloads Chromium), then continue without the browser.

## Rules

- Open only pages the access rules in `sources.md` allow. A browser doesn't change what's allowed.
- Check every page right after `open`, before any `wait --load` (a `networkidle` wait never returns on a challenge page and leaves the daemon stuck):

  ```bash
  agent-browser --session deal-check-ro get title
  agent-browser --session deal-check-ro get url
  ```

  If the title is "Just a moment…" or asks you to verify you're human, or the URL starts with `chrome-error://` (connection failed), stop using that source for this run. Don't wait it out, click through or solve it.
- In practice a headless agent-browser gets a challenge or connection error on compari.ro, PC Garage and Flanco, and an HTTP/2 error on Altex (tested November 2026). The skill doesn't open them, and uses istoric-preturi's prices for those stores. The only exception is a site the user names as an extra site: one try.
- Accept no cookie banners beyond what's needed to read the page. Prefer "Refuz"/"Reject" when it blocks content.
- Close the session at the end: `agent-browser --session deal-check-ro close`.

## Read a product page (JSON-LD)

```bash
agent-browser --session deal-check-ro open '<product url>'
# check title and URL first (see Rules), then:
agent-browser --session deal-check-ro wait --load networkidle
cat <<'JS' | agent-browser --session deal-check-ro eval --stdin
JSON.stringify([...document.querySelectorAll('script[type="application/ld+json"]')]
  .map(s => { try { return JSON.parse(s.textContent) } catch { return null } })
  .flatMap(x => !x ? [] : Array.isArray(x) ? x : x['@graph'] || [x])
  .filter(x => x && [].concat(x['@type']).includes('Product'))
  .map(p => { const o = [].concat(p.offers || [])[0] || {};
    return { name: p.name, brand: p.brand && (p.brand.name || p.brand), mpn: p.mpn,
             gtin: p.gtin13 || p.gtin12 || p.gtin8 || p.gtin, sku: p.sku,
             price: o.price || o.lowPrice, availability: o.availability, condition: o.itemCondition } }))
JS
```

## Read history from a Chart.js chart

All three history sources draw with Chart.js. This script works on each of them and returns `[{canvas, series: [{label, points: [[date, price], …]}]}]`:

```bash
cat <<'JS' | agent-browser --session deal-check-ro eval --stdin
(() => {
  const C = window.Chart;
  if (!C || !C.instances) return JSON.stringify({ error: "no Chart.js chart on page" });
  const day = (v) => {
    if (v instanceof Date) return v.toISOString().slice(0, 10);
    if (typeof v === "number") return new Date(v).toISOString().slice(0, 10);
    return String(v);
  };
  return JSON.stringify(Object.values(C.instances).map((ch) => {
    const labels = ch.data.labels || [];
    return {
      canvas: ch.canvas && ch.canvas.id,
      series: ch.data.datasets.map((d) => ({
        label: d.label,
        points: d.data.map((p, i) => {
          if (p == null) return null;
          if (typeof p === "object") return p.y == null ? null : [day(p.t ?? p.x), Number(p.y)];
          return [day(labels[i]), Number(p)];
        }).filter(Boolean),
      })),
    };
  }));
})()
JS
```

Per-site steps before running it:

| Site | Steps | What you get |
|---|---|---|
| istoric-preturi.info | `open` product page, check title, `wait --load networkidle`. Read the offers with `get text body` and the JSON-LD with the script above from this same page; don't fetch it again with curl | `#pricesChart`, one series per store (label = store name). Dates like `02 Noi 2026` use Romanian month abbreviations: Ian Feb Mar Apr Mai Iun Iul Aug Sep Oct Noi Dec |
| compari.ro | `open` product page, check title (usually a challenge in headless mode), `wait --load networkidle` | `#chartnormal`, series `preţ minim` and `preţ mediu`, ISO dates |
| pricy.ro | `open` product page, then click the "Istoric preț" tab: `agent-browser --session deal-check-ro find first 'a[href="#price-history"]' click`, `wait 3000`. For the full range also click "Tot": `agent-browser --session deal-check-ro find text "Tot" click --exact`, `wait 2000` | `#priceHistoryCanvas`, series `Cel mai mic preț` and `Preț mediu`, ISO dates. Default range is 1 year |

`eval` prints the script's return value as a JSON string literal (`"[{\"canvas\":…"`), so decode it twice: once for the string, once for the data.

If the script returns `no Chart.js chart`, wait 2 to 3 seconds and run it once more. If it still fails, report history as unavailable for that source.

## Read offers from a rendered page

When a comparator's offers are only in rendered HTML (compari.ro):

```bash
agent-browser --session deal-check-ro get text body
```

Then pick out each offer: store domain, offer title (check it names the same model and variant), price, delivery cost if shown. `snapshot -i` is noisier; prefer text.
