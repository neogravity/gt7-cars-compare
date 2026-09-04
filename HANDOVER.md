# GT7 Gr.1–4 Compare Tool — Handover Notes

## What this is

`gt7-compare.html` is a single-file, self-contained HTML/CSS/JS tool that lets you sort, filter, and side-by-side compare all 121 racing-class cars in Gran Turismo 7 (Gr.1: 25, Gr.2: 10, Gr.3: 52, Gr.4: 34). No build step, no dependencies — open it directly in a browser.

Features: sortable columns, group/manufacturer/drivetrain/aspiration filters, PP/power range sliders, search, thumbnail images, and a compare tray (pin up to 4 cars → side-by-side modal).

## Data source (important — read before touching data)

All spec data (displacement, drivetrain, max power w/ RPM, max torque w/ RPM, weight, aspiration, dimensions) comes directly from **Sony/Polyphony's own `gran-turismo.com` car list**, not a third-party site. Getting it required a specific technique — worth understanding before extending this:

1. `https://www.gran-turismo.com/us/gt7/carlist/id/car{ID}` is a **JS-rendered SPA**. A plain HTTP fetch only returns an empty shell with meta tags (title, og:image) — no spec data. You need something that actually executes JavaScript (a real browser / headless browser / Playwright / Puppeteer) to see the rendered content.
2. The entire car database is shipped as **one static JS data bundle**, not fetched per-car. Find it via the browser's `performance.getEntriesByType('resource')` while on any carlist page — look for a file matching `assets/cars.{locale}-*.js` (e.g. `cars.us-8tX_Xh6V.js`). **The hash in the filename changes on redeploys**, so re-discover it if it 404s.
3. That file is `var e={car102:{...}, car1027:{...}, ...};export{e as Cars};` — a plain JS object keyed by `car{ID}`, fetchable directly (`fetch(url).then(r=>r.text())`) and `eval`-able once you strip the `export` statement. It contains **every one of the 574 GT7 cars**, all categories, in one shot.
4. Fields per car object:
   ```
   id, nameLong, nameShort, manufacturerId, countryId, carClass  // "Gr.1".."Gr.4", "Gr.N" (road/street), "Gr.B", "Gr.2", etc.
   driveTrain, driveTrain_v         // "FR"/"MR"/"RR"/"FF"/"4WD"
   aspirationLong, aspirationShort, aspiration_v
   displacement, displacement_v     // string like "5935 cc" or "- cc" if not disclosed; note rotary engines show like "654x4 cc"
   maxPower, power, power_v         // maxPower = "693 HP / 9200 rpm" (includes RPM!); power_v = numeric HP only
   maxTorque, torque, torque_v      // maxTorque = "439.4 ft-lb / 7500 rpm"; NATIVE UNIT IS ft-lb, not Nm
   weight, weight_v                 // "2094 lbs."; NATIVE UNIT IS lbs, not kg
   length, length_v, width, width_v, height, height_v   // in inches
   performancePoint                 // "PP 889.97"
   ```
   **Units are all imperial natively.** This tool converts to metric (kg, cm, Nm) for display alongside the imperial originals — don't assume Nm/mm/kg are the source of truth, inches/lbs/ft-lb are.
5. To map a car **name → ID**, the main list page (`/us/gt7/carlist/`) has anchor tags for every car: `a[href*="/carlist/id/car"]`, `innerText` = display name, href contains the numeric ID. Build this map once, then look up IDs for whatever cars you need out of the big data bundle.
6. **Images:** `https://www.gran-turismo.com/common/dist/gt7/carlist/car_thumbnails/car{ID}.png` (320×180, confirmed working) is the primary source used in this tool. Fallback: `https://www.gran-turismo.com/common/dist/gt7/carlist/og_images/car{ID}_1_01.jpg` (larger hero image, also official).
7. **Price is NOT on gran-turismo.com's car list.** This tool sources `price` from a secondary community database, `dg-edge.com` (branded "GT ENGINE" data), which also independently confirmed drivetrain/power/weight/PP for cross-validation during development. If price ever looks stale, that's the source to revisit — it's not from Polyphony directly.

### Gotchas hit during development (avoid repeating)
- A generic `fetch()` to gran-turismo.com image URLs from within page JS occasionally hung indefinitely even with an `AbortController` — if that happens, just `navigate` the browser tab directly to the image URL and read `document.images[0].naturalWidth` instead of fetching.
- When pulling long JSON strings out of a browser tool's JS-exec result, output gets **silently truncated around ~995 characters** with no error — no `[TRUNCATED]` marker in some cases. Pull in explicit `.slice(start, start+900)` windows and verify total length up front (`str.length`) so you know exactly how many chunks to request. Double-check chunk boundaries when reassembling (off-by-one/dropped-character errors are easy to introduce when hand-copying chunked output — validate with `JSON.parse` and fix seams if it throws).

## File structure

Currently just one file: `gt7-compare.html`. Everything (data, CSS, JS) is inline.

### `RAW` array (top of the `<script>` block)
Each row is a fixed-order array — **not** an object — for compactness:
```
[0]  name              (string, display name, may omit manufacturer)
[1]  manufacturer       (string)
[2]  group              (string, "GR.1".."GR.4")
[3]  drivetrain         (string, "FR"/"MR"/"RR"/"FF"/"4WD")
[4]  maxPower_HP        (number)
[5]  weight_kg          (number, converted from lbs)
[6]  aspiration         (string, "NA"/"TC"/"SC")
[7]  PP                 (number)
[8]  price_credits      (number, from dg-edge.com — see caveat above)
[9]  displacement_cc    (string or null; null when GT7 doesn't disclose it, e.g. many VGT/Gr.3 concept cars)
[10] maxTorque_Nm       (number, converted from ft-lb)
[11] length_mm          (number, converted from inches)
[12] width_mm           (number, converted from inches)
[13] height_mm          (number, converted from inches)
[14] officialCarId      (string — the numeric ID used in gran-turismo.com URLs, e.g. "3394")
[15] powerPeak_rpm      (number)
[16] torquePeak_rpm     (number)
[17] weight_lb          (number, native unit)
[18] length_in          (number, native unit)
[19] width_in           (number, native unit)
[20] height_in          (number, native unit)
[21] maxTorque_ftlb     (number, native unit)
```
`CARS` (below `RAW`) maps this into named objects for the rest of the app to use.

### Key functions
- `getFiltered()` / `render()` — filter + sort + redraw the table
- `imgUrl(c)` / `imgFallbackUrl(c)` — build the two candidate thumbnail URLs from `carId`
- Compare tray/modal logic near the bottom — pins up to 4 cars via checkboxes, builds a side-by-side spec table on demand

## Known limitations / open items

1. **"RX-VISION GT3 CONCEPT Stealth Model"** isn't listed as a separate entry on the official car-list site (it's a limited event-reward livery variant). It currently reuses the base RX-VISION GT3 CONCEPT's specs and image — specs are confirmed identical (same PP, power, weight), but if Polyphony ever gives it distinct stats this row will be stale.
2. **No live-update mechanism.** This is a snapshot as of the Spec III-era patch (early Sept 2026, per PP values like the AMG GT3 '20 at PP 731.13). GT7 gets periodic balance-of-performance (BOP) patches that shift PP and sometimes power/weight — there's no automation here to detect and re-pull when that happens.
3. **Scope is Gr.1–4 only** (racing classes). The same `cars.{locale}-*.js` bundle + ID-mapping technique would extend cleanly to Gr.B (rally), Sport (road cars), and Super Formula if ever needed — just rebuild the target name list and re-run the extraction steps above.
4. **Price accuracy** — see data source note above; it's the one field not sourced from Polyphony directly.

## Suggested next steps

- If continuing in Claude Code: consider splitting the single HTML file into `index.html` / `styles.css` / `data.js` / `app.js` for easier diffs, though the single-file format was intentional for portability up to now.
- A small Node/Playwright script that re-runs the extraction steps above (steps 1–6) would let this refresh automatically after GT7 patches, rather than requiring manual re-extraction via a chat session.
- Consider persisting `RAW` as a separate `cars.json` so the dataset can be updated independently of the UI code.
