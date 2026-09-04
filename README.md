# GT7 Compare — Gr.1 to Gr.4

A single-file, zero-dependency web tool to **sort, filter, and side-by-side compare every racing-class car in Gran Turismo 7** (Groups 1–4). Open it in any modern browser — no build step, no install, no server.

- **121 cars** — Gr.1 (25), Gr.2 (10), Gr.3 (52), Gr.4 (34)
- Specs pulled directly from **Sony/Polyphony's own `gran-turismo.com` data**, not a third-party mirror
- **List** and **Grid** views, per-column show/hide, rich filtering, and a pin-to-compare tray

> ⚠️ Fan project. Not affiliated with, endorsed by, or sponsored by Sony Interactive Entertainment or Polyphony Digital. See [Data & attribution](#data--attribution).

---

## Table of contents

- [Quick start](#quick-start)
- [Features](#features)
- [Usage](#usage)
- [Project structure](#project-structure)
- [Data specification](#data-specification)
- [Data sources & provenance](#data-sources--provenance)
- [Updating the dataset](#updating-the-dataset)
- [Architecture notes](#architecture-notes)
- [Known limitations](#known-limitations)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [Data & attribution](#data--attribution)
- [License](#license)

---

## Quick start

```bash
# Clone, then just open the file — that's it.
open index.html          # macOS
# or: xdg-open index.html # Linux
# or double-click it in your file manager
```

There is no toolchain. Everything (data, CSS, JavaScript) is inline in one HTML file. The only network requests it makes at runtime are for car thumbnail images from `gran-turismo.com` and the Google Fonts stylesheet; it works offline apart from those.

---

## Features

### Views
- **List view** — a dense, sortable spec table (the classic layout).
- **Grid view** — thumbnail cards, **8 per row** on desktop (responsive down to 6 / 4 / 2 on narrower screens), with the selected attributes stacked beneath each image.
- The chosen view is remembered across reloads (`localStorage`).

### Column show/hide
- A **Columns** dropdown lets you toggle any of the 14 data columns on or off (e.g. hide *Max Power* or *Price*).
- The choice applies to **both** List and Grid views and is **persisted** to `localStorage`.

### Filtering & search
- **Group** pills (Gr.1–4), **Manufacturer**, **Drivetrain**, and **Aspiration** filters.
- **PP** and **Power (HP)** upper-bound range sliders.
- Free-text **search** across model name and manufacturer.
- One-click **Reset filters**.

### Sorting
- Click any List-view column header to sort ascending/descending. Sort state carries into Grid view.

### Compare
- Check up to **4 cars** to pin them to the compare tray, then open a **side-by-side modal** with a full spec breakdown (including a computed **power-to-weight** row).

### Units
- Displays **metric** figures (kg, cm, Nm) converted from the site's **native imperial** units (lbs., in., ft-lb), with the originals shown alongside for reference.

---

## Usage

1. **Pick a view** with the List / Grid toggle above the table.
2. **Narrow the field** with the group pills, dropdown filters, sliders, or search box.
3. **Choose your columns** via the *Columns* menu — hide anything you don't care about.
4. **Sort** by clicking a column header (List view).
5. **Compare**: tick the checkbox on 2–4 cars, then hit **Compare selected →** in the bottom tray.

Your view and column preferences are saved locally in your browser, so they persist between sessions on the same machine.

---

## Project structure

```
.
├── index.html         # The application — UI, CSS, and app logic
├── cars.js            # The dataset (window.RAW), separated for easy updates
├── README.md          # This file
├── HANDOVER.md        # Deep-dive developer notes on how the data was extracted
├── LICENSE            # MIT (code) + game-data attribution note
└── .gitignore
```

The app is [`index.html`](index.html); the data lives in [`cars.js`](cars.js), which assigns `window.RAW` and is loaded via a `<script src="cars.js">` tag before the app code. This keeps the two concerns separate while staying **fully static and dependency-free** — it works both when served over HTTP ([GitHub Pages](https://neogravity.github.io/gt7-cars-compare/) serves `index.html` as the site root) *and* when you just double-click `index.html` from disk (`file://`). A `.js` data file is used rather than `.json` precisely so `file://` keeps working — browsers block `fetch()` of a local JSON file, but a `<script src>` loads fine.

---

## Data specification

The dataset lives in [`cars.js`](cars.js) as **`window.RAW`** — an array where each car is a **fixed-order array** (not an object) for compactness. In `index.html`, `CARS` maps these into named objects for the rest of the app.

| Index | Field | Type | Notes |
|------:|-------|------|-------|
| 0  | `name` | string | Display name (may omit manufacturer) |
| 1  | `manufacturer` | string | |
| 2  | `group` | string | `"GR.1"`–`"GR.4"` |
| 3  | `drivetrain` | string | `"FF"` / `"FR"` / `"MR"` / `"RR"` / `"4WD"` |
| 4  | `maxPower_HP` | number | Peak power, horsepower |
| 5  | `weight_kg` | number | Converted from lbs |
| 6  | `aspiration` | string | `"NA"` / `"TC"` / `"SC"` / `"EV"` |
| 7  | `PP` | number | Performance Points |
| 8  | `price_credits` | number | In-game credits — sourced from dg-edge.com (see caveat) |
| 9  | `displacement_cc` | string \| null | `null` when GT7 doesn't disclose it (many VGT / concept cars) |
| 10 | `maxTorque_Nm` | number | Converted from ft-lb |
| 11 | `length_mm` | number | Converted from inches |
| 12 | `width_mm` | number | Converted from inches |
| 13 | `height_mm` | number | Converted from inches |
| 14 | `officialCarId` | string | Numeric ID used in `gran-turismo.com` URLs (e.g. `"3394"`) |
| 15 | `powerPeak_rpm` | number | |
| 16 | `torquePeak_rpm` | number | |
| 17 | `weight_lb` | number | **Native unit** |
| 18 | `length_in` | number | **Native unit** |
| 19 | `width_in` | number | **Native unit** |
| 20 | `height_in` | number | **Native unit** |
| 21 | `maxTorque_ftlb` | number | **Native unit** |

> **Imperial is the source of truth.** Weight, torque, and dimensions are natively imperial on `gran-turismo.com` (lbs, ft-lb, inches). The metric values in the table are conversions computed for display — don't treat kg/Nm/mm as canonical.

---

## Data sources & provenance

All specification data — displacement, drivetrain, max power (with RPM), max torque (with RPM), weight, aspiration, and dimensions — comes **directly from Polyphony's `gran-turismo.com` car list**.

Key facts about the source (full detail in [`HANDOVER.md`](HANDOVER.md)):

1. The car-list pages are a **JS-rendered SPA** — a plain HTTP fetch returns only an empty shell. You need a real/headless browser to see rendered content.
2. The whole car database ships as **one static JS bundle** — `assets/cars.{locale}-*.js` — keyed by `car{ID}`, containing **all ~574 GT7 cars**. The hash in the filename changes on redeploys, so it must be re-discovered when it 404s.
3. **Thumbnails**: `car_thumbnails/car{ID}.png` (320×180, primary); `og_images/car{ID}_1_01.jpg` (larger hero, fallback).

**Price is the one exception.** GT7's car list does not publish in-game prices, so `price` is sourced from the **dg-edge.com** community database ("GT ENGINE" data), which also independently corroborated drivetrain/power/weight/PP during development.

**Snapshot version:** the data reflects the **Spec III-era patch (early September 2026)**. GT7 receives periodic Balance-of-Performance (BOP) patches that adjust PP and occasionally power/weight — there is no auto-refresh here.

---

## Updating the dataset

When a BOP patch lands (or to add more classes), re-run the extraction:

1. Open any `gran-turismo.com` car-list page in a browser.
2. Find the current data bundle via `performance.getEntriesByType('resource')` → look for `assets/cars.{locale}-*.js`.
3. Fetch it as text, strip the trailing `export{...}` statement, and `eval` the `var e={...}` object.
4. Build a **name → ID** map from the list page's anchors: `a[href*="/carlist/id/car"]` (text = display name, href = ID).
5. Pull the fields you need per car and rebuild the `RAW` rows.
6. Refresh `price` from dg-edge.com if desired.

See [`HANDOVER.md`](HANDOVER.md) for the gotchas (silent ~995-char truncation of long JS-exec output, hanging image fetches, etc.).

---

## Architecture notes

- **`getFiltered()` / `render()`** — filter + sort, then dispatch to `renderList()` or `renderGrid()`.
- **Column visibility** — every `<th>`/`<td>` and every grid card field carries a `col-<id>` class; `applyColVisibility()` toggles a `.col-hidden` (`display:none !important`) utility across both views. The `!important` matters: the grid card's `.card-specs > div { display:flex }` rule otherwise out-specifies a plain `.col-hidden`.
- **`imgUrl()` / `imgFallbackUrl()`** — build the two candidate thumbnail URLs from `carId`, with an `onerror` chain that falls back to the hero image and finally a "no image" placeholder.
- **Compare tray/modal** — pins up to 4 cars via checkboxes and builds the side-by-side spec table on demand.
- **Persistence** — view mode (`gt7-view`) and hidden columns (`gt7-hidden-cols`) are stored in `localStorage`; all storage access is wrapped in `try/catch` so the app degrades gracefully where storage is unavailable (e.g. `data:`-URL sandboxes).

---

## Known limitations

1. **"RX-VISION GT3 CONCEPT Stealth Model"** isn't a separate entry on the official car list (it's an event-reward livery variant); it reuses the base car's specs and image. Specs are confirmed identical today, but could drift if Polyphony ever differentiates them.
2. **No live-update mechanism.** The data is a manual snapshot; BOP patches require a manual re-pull.
3. **Scope is Gr.1–4 only.** The same extraction technique extends cleanly to Gr.B (rally), Sport (road cars), and Super Formula.
4. **Price accuracy** depends on the dg-edge.com community source, not Polyphony directly.

---

## Roadmap

Ideas, not commitments:

- A small Node/Playwright script to automate the extraction and refresh after patches.
- Split the dataset into a standalone `cars.json` so data can be updated independently of the UI.
- Optionally split the single HTML into `index.html` / `styles.css` / `data.js` / `app.js` for easier diffs (single-file portability is currently intentional).
- Extend coverage to Gr.B / Sport / Super Formula.

---

## Contributing

Issues and PRs welcome. Because it's a single file:

- Keep changes inline and dependency-free.
- If you touch the dataset, follow the [Data specification](#data-specification) field order exactly and validate with `JSON.parse` after any hand-editing.
- Preserve the imperial-native values (indices 17–21) as the source of truth; recompute metric display values from them.

---

## Data & attribution

Vehicle specifications, names, and thumbnail images originate from **Gran Turismo 7**, © **Sony Interactive Entertainment / Polyphony Digital**. In-game pricing is sourced from the **dg-edge.com** community database. All such data and imagery remain the property of their respective owners and are used here for a **non-commercial fan reference tool**. This project is **not affiliated with, endorsed by, or sponsored by** Sony or Polyphony Digital.

---

## License

Source code is released under the **[MIT License](LICENSE)**. The MIT grant covers the code only — see the [Data & attribution](#data--attribution) note above regarding game data and imagery.
