# Address → Lat/Lon Geocoder

A single-file web app that bulk-geocodes addresses from a CSV or XLSX file and exports
the result with `lat`, `lon`, `match_level`, and `matched_address` columns added.
Works for any country (pick from a dropdown; default Israel, Hebrew supported, e.g.
`דיזנגוף 50, תל אביב`).

## Usage

1. Open `index.html` in any modern browser (no install, no server needed).
2. Choose or drag in a `.csv`, `.xlsx`, or `.xls` file. Row 1 must contain a column
   whose header is `address` (case-insensitive). Other columns are kept untouched.
3. Pick the **geocoder**:
   - **Geoapify** (default) — higher coverage; needs a free API key (see below).
   - **Nominatim / OpenStreetMap** — free, no key, but rate-limited and lower coverage.
4. Pick the **country** (or "No restriction" to let it auto-detect from the text).
5. Optionally tick **Fall back to approximate match** — off by default, so only exact
   matches are returned and anything else is left blank. Tick it to let messy addresses
   degrade to a street- or city-level result instead.
6. Click **Geocode** and watch the progress bar.
7. Click **Download result** — you get the same format you uploaded (CSV→CSV,
   XLSX→XLSX) with the new columns appended after the original ones.
   The output file is named `<original>_geocoded.<ext>`.

## Output columns

- `lat` / `lon` — coordinates (blank if nothing matched).
- `match_level` — how the match was found, i.e. how much to trust it:
  `exact` (address as-is) › `cleaned` (noise tokens removed) › `street+city` ›
  `city` (only the city matched — approximate) › `none` (no match).
- `matched_address` — the full address the geocoder returned, for eyeballing.

## Handling messy addresses

Nominatim's search requires *all* tokens to match, so extra references — landmarks,
building/gate names, floors, malls (e.g. `שער מילאנו, קומת 1, מתחם חדש`) — make a
lookup fail. The app retries a **cascade** of progressively simpler variants and records
which one hit in `match_level`. Sort/filter by that column to review low-confidence rows.

## How it works

- Parsing/writing of CSV and XLSX is done with [SheetJS](https://sheetjs.com) (loaded
  from a CDN — an internet connection is required).
- Geocoding goes through one of two providers, selected in the UI:
  - [Geoapify](https://www.geoapify.com/) (`/v1/geocode/search`) — default. The country sets
    `filter=countrycode:` and `lang`.
  - [Nominatim](https://nominatim.openstreetmap.org) (OpenStreetMap) — free. The
    country sets `countrycodes` and the `accept-language` of returned labels.
  Both go through the same `geocodeCascade` retry logic; only the per-request query function
  and throttle differ.
- **Provider fallback:** when **Nominatim** is selected and an API key is supplied, any
  address OpenStreetMap can't find is automatically retried through Geoapify. Rows resolved
  this way are counted as "rescued via Geoapify" in the summary. If the fallback key is bad,
  the fallback is disabled for the rest of the run (Nominatim keeps working) with a warning;
  a bad key on the *primary* provider stops the run immediately.

## Geoapify API key

Geoapify needs a key (free tier: ~5 requests/second, 3,000 lookups/day). Get one at
[geoapify.com](https://www.geoapify.com/), then paste it into the **API key** field in the app.

- The key is stored **only in your browser** (`localStorage`, if "Remember on this browser"
  is ticked) and is sent only to Geoapify. It is **never** written to the code or the repo.
- **Do not hardcode the key in `index.html`** — this is a public client-side app, so anything
  in the source is visible to every visitor and would land in git history.
- If you publish this on GitHub Pages, restrict your key in the Geoapify dashboard to your
  Pages origin (allowed origins / HTTP referrers) so others can't spend your quota.

## Notes & limits

- **Rate limit:** Nominatim allows ~1 request/second, so the app throttles to that
  (~1 second per attempt). Geoapify's free tier allows ~5/second, so it throttles far less
  (~0.25s). A hard-to-match address tries several variants (each one request). Identical
  queries are cached and only looked up once.
- **OSM coverage:** some streets (notably in Israel) simply aren't in OpenStreetMap, so
  they degrade to `city` level or `none` no matter how the address is cleaned. For higher
  coverage you'd need a paid provider (e.g. Google Maps); the `geocodeCascade` function is
  isolated to make that swap easy.
- CSV output includes a UTF-8 BOM so Hebrew text displays correctly when reopened in Excel.
