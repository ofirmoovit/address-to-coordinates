# Address → Lat/Lon Geocoder

A single-file web app that bulk-geocodes addresses from a CSV or XLSX file and exports
the result with `lat`, `lon`, `match_level`, and `matched_address` columns added.
Works for any country (pick from a dropdown; default Israel, Hebrew supported, e.g.
`דיזנגוף 50, תל אביב`).

## Usage

1. Open `index.html` in any modern browser (no install, no server needed).
2. Choose or drag in a `.csv`, `.xlsx`, or `.xls` file. Row 1 must contain a column
   whose header is `address` (case-insensitive). Other columns are kept untouched.
3. Pick the **country** (or "No restriction" to let it auto-detect from the text).
4. Optionally tick **Fall back to approximate match** — off by default, so only exact
   matches are returned and anything else is left blank. Tick it to let messy addresses
   degrade to a street- or city-level result instead.
5. Click **Geocode** and watch the progress bar.
6. Click **Download result** — you get the same format you uploaded (CSV→CSV,
   XLSX→XLSX) with the new columns appended after the original ones.
   The output file is named `<original>_geocoded.<ext>`.

## Output columns

- `lat` / `lon` — coordinates (blank if nothing matched).
- `match_level` — how the match was found, i.e. how much to trust it:
  `exact` (address as-is) › `cleaned` (noise tokens removed) › `street+city` ›
  `city` (only the city matched — approximate) › `none` (no match).
- `matched_address` — the full address Nominatim returned, for eyeballing.

## Handling messy addresses

Nominatim's search requires *all* tokens to match, so extra references — landmarks,
building/gate names, floors, malls (e.g. `שער מילאנו, קומת 1, מתחם חדש`) — make a
lookup fail. The app retries a **cascade** of progressively simpler variants and records
which one hit in `match_level`. Sort/filter by that column to review low-confidence rows.

## How it works

- Parsing/writing of CSV and XLSX is done with [SheetJS](https://sheetjs.com) (loaded
  from a CDN — an internet connection is required).
- Geocoding uses the free [Nominatim](https://nominatim.openstreetmap.org)
  (OpenStreetMap) service. The selected country sets `countrycodes` and the
  `accept-language` of returned labels.

## Notes & limits

- **Rate limit:** Nominatim allows ~1 request/second, so the app throttles to that.
  A hard-to-match address tries several variants (each one request), so budget ~1 second
  per attempt. Identical queries are cached and only looked up once.
- **OSM coverage:** some streets (notably in Israel) simply aren't in OpenStreetMap, so
  they degrade to `city` level or `none` no matter how the address is cleaned. For higher
  coverage you'd need a paid provider (e.g. Google Maps); the `geocodeCascade` function is
  isolated to make that swap easy.
- CSV output includes a UTF-8 BOM so Hebrew text displays correctly when reopened in Excel.
