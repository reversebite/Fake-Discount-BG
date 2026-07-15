# Fake Discount Bulgaria — Catch Faked Sales

> Chrome / Edge extension

A Manifest V3 Chrome/Edge extension with adapters for 20 Bulgarian e-commerce sites (Emag.bg, Ozone.bg, Notino.bg, Technopolis.bg, Technomarket.bg, Zora.bg, Ardes.bg, Plesio.bg, Aboutyou.bg, Answear.bg, Decathlon.bg, dm-drogeriemarkt.bg, Fashiondays.bg, Lillydrogerie.bg, Mr-bricolage.bg, Obuvki.bg, Praktiker.bg, Sopharmacy.bg, Sportdepot.bg, eBag.bg). It records a daily EUR price snapshot when you visit a supported, enabled product page and renders local price-history charts plus heuristic deal signals directly on the page.

> **Pre-release status:** the 15 July 2026 audit found public Supabase access,
> adapter correctness bugs, and reproduced storage/import races that must be
> fixed before distribution. See [composer 2.5 multi agent audit.md](composer%202.5%20multi%20agent%20audit.md).

**Repository:** https://github.com/victortserovski/Fake-Discount-BG
**Privacy policy:** [PRIVACY.md](PRIVACY.md)

## Features

- **Visit-triggered Tracking**: A valid purchasable price is recorded automatically when you visit an enabled, supported product page (no manual watchlist needed; one local snapshot per product per day)
- **Integrated Display**: Price graph and history-based deal analysis displayed directly on product pages
- **Verdict System**: Shows the heuristic states "Fake discount", "Real deal", "Volatile price", "Stable price", or "Tracking" with the observations behind the result
- **Extension Badge**: Per-tab icon badge — "!" for fake discounts, "✓" for real deals, 🎯 when a price target is hit (target badge overrides the verdict badge on that tab)
- **Price Targets**: Set a target on any product; the chart shows a horizontal target line, the popup marks active targets, and the current tab badge/pill changes when a visited price reaches the target. There is no background polling or browser notification.
- **Clickable Product List**: Click any product in the popup to open its page
- **Bilingual Support**: Bulgarian (default) and English, with localized date formatting in the chart
- **Export/Import**: Export and import price-history product records as JSON with field-level validation (settings and price targets are not included; import is not transactional)
- **EAN/GTIN tracking**: When a product's barcode is exposed and recognized, it is stored alongside price history and can be searched or copied from the popup. Saved fixtures confirm extraction on **Notino, Obuvki, Zora, dm-drogeriemarkt, and Answear**, with occasional description-based coverage on **Praktiker** and **Mr.Bricolage**. Technomarket fixtures expose EAN attributes that the current generic extractor does not yet read. Cross-store matching is a future feature.
- **Last-seen indicator**: Each product card shows when it was last refreshed ("updated today / yesterday / N days ago") so stale entries are easy to spot
- **Keyboard shortcut**: Press `Ctrl+Shift+F` (or `⌘+Shift+F` on Mac) to open the popup. Customisable at `chrome://extensions/shortcuts`
- **Persisted filters**: Site-filter chips and sort selection survive popup close/reopen (the search box is intentionally cleared each time)

## Installation

1. Download or clone this repository
2. Open Chrome/Edge and navigate to `chrome://extensions/` (or `edge://extensions/`)
3. Enable "Developer mode" (toggle in top right)
4. Click "Load unpacked"
5. Select the extension directory

## How It Works

1. When you visit a product page on any supported site (Emag, Ozone, Notino, Technopolis, Technomarket, Zora, Ardes, Plesio, Aboutyou, Answear, Decathlon, dm-drogeriemarkt, Fashiondays, Lillydrogerie, Mr-bricolage, Obuvki, Praktiker, Sopharmacy, Sportdepot, eBag), the extension automatically:
   - Extracts product information (ID, price, title)
   - Stores or updates one local price snapshot for that product and local day
   - Analyzes local price history for potentially misleading or favorable pricing signals
   - Displays a price graph widget on the product page

2. The extension tracks:
   - Current price vs. historical prices
   - Claimed "original price" vs. actual historical maximum
   - Price trends over 30-day windows
   - Overall pricing stability/volatility

3. Verdict System:
   - **FAKE DISCOUNT** (red): A warning signal — the claimed "original" price exceeds observed history, or the current price is well above the 30-day low. The second rule can fire even when no seller discount is shown.
   - **REAL DEAL** (green): A favorable observed price — near the tracked low, or (when no seller "was" price is shown) materially below the historical average
   - **VOLATILE PRICE** (orange): Price has fluctuated by 8%+ across the last 30 days — wait for a low point
   - **STABLE PRICE** (yellow): Price has been confirmed stable over 7+ price observations AND the 30-day range is tight (< 8% of average)
   - **TRACKING** (gray): Still gathering data — fewer than 7 price observations (`insufficientData` reason), or 7+ observations with no rule matched yet (`noPatternMatch` reason)

   History-based verdict reasons report the relevant observation count and
   tracked span. The insufficient-data state instead reports progress toward
   the seven-observation threshold.

## Storage

- Uses Chrome local storage (limit ~10MB)
- Per-product keys for O(1) read/write performance
- Per-product writes are serialized during ordinary tracking; the current
  audit found separate migration and clear-all races that still need repair
- **Recorded daily snapshots are not compressed or automatically evicted.**
  They remain until manual deletion, extension removal, or Chrome's local
  storage quota is reached.
- Storage usage shown with adaptive precision (e.g. "0.03%") in the popup
- No automatic deletion — products can be removed individually, by the manual
  90-day cleanup, or with Clear all history

## Popup

Click the extension icon. The popup is organised into three tabs plus a "↗"
link in the header that re-opens the same UI as a full Chrome tab (handy when
you have many tracked products).

**Products tab** — search by name or EAN, filter by site (multi-select chips
for all 20 supported stores: EMAG, OZONE, NOTINO, TECHNOPOLIS, TECHNOMARKET,
ZORA, ARDES, PLESIO, ABOUT YOU, ANSWEAR, DECATHLON, DM, FASHION DAYS, LILLY,
MR.BRICOLAGE, OBUVKI, PRAKTIKER, SOPHARMACY, SPORT DEPOT, EBAG), sort by recently
visited / price / targets-first. Each card
shows thumbnail, title, EAN (when known, click to copy), "last seen" line,
price, trend (↑ higher / ↓ lower / → same — only shown once there's a
prior price to compare against), site badge, and a 🎯 row when a price
target is set.

**Settings tab**
- Change language (Bulgarian/English)
- **Track prices on each supported site** — a master toggle per store
  (Emag, Ozone, Notino, Technopolis, Technomarket, Zora, Ardes, Plesio,
  Aboutyou, Answear, Decathlon, dm, Fashiondays, Lilly, Mr-bricolage,
  Obuvki, Praktiker, Sopharmacy, Sportdepot, eBag). When off, no tracking,
  no widget, and no Supabase uploads for that site.
- **Show chart on product pages** — visibility toggle. When off, prices are
  still tracked silently in the background and uploaded to the shared dataset;
  the chart just doesn't appear.
- **About** subsection — current version (read live from `manifest.json`),
  contact email, keyboard-shortcut hint and link to customise it

**Data tab**
- Storage usage bar + tracked-product count
- Export/import price-history product records as JSON (settings and price targets are not included; malformed product rows are skipped)
- Delete products not seen for 90+ days, or clear all indexed history

A persistent footer at the bottom of every tab shows total tracked count and
storage usage at a glance.

## Architecture

- **Manifest V3** with content scripts and background service worker
- `content/` - Site-specific content scripts (emag.js, ozone.js, notino.js, technopolis.js, technomarket.js, zora.js, ardes.js, plesio.js, aboutyou.js, answear.js, decathlon.js, dm.js, fashiondays.js, lilly.js, bricolage.js, obuvki.js, praktiker.js, sopharmacy.js, sportdepot.js, ebag.js) with shared base (content-base.js)
- `background/` - Service worker for message handling, storage, and price analysis
- `ui/` - SVG-based chart rendering (advanced-chart.js) and widget UI (price-graph-widget.js)
- `i18n/` - Bulgarian and English translation files
- `popup/` - Extension popup with settings and product list
- `utils/` - PriceStorageManager with per-product keys and migration support; Supabase best-effort, one-way upload utility (see [PRIVACY.md](PRIVACY.md))

## Development

The widget UI scripts run in the content script isolated world (not injected into the host page). This ensures they work on sites with strict Content Security Policies like Ozone.bg. SPA navigation is detected via `chrome.tabs.onUpdated` messages from the background service worker.

### Cloud sync (enabled in the public build)

`utils/supabase-sync.js` ships with populated `SUPABASE_URL` and
`SUPABASE_ANON_KEY` constants pointing at a developer-controlled
Supabase project. Each recorded daily snapshot is submitted best-effort,
one-way to a `price_history` Postgres table — see
[PRIVACY.md](PRIVACY.md) for the exact field list and how the data is
used. The extension does not download that pooled dataset: verdicts and
charts use local history only. Local `chrome.storage` remains the source
of truth; the network push never blocks the widget render. Upload is deduplicated per
`(device_id, product_id, observed_date)` so repeated visits within the
same day don't bloat the dataset.

The current database policy exposes reads through the bundled public anon
key. This must be replaced with a controlled write path and direct table
access must be revoked before public distribution; see the audit report.

Developers cloning this repo for their own fork can blank both constants
to disable the upload (it then becomes a silent no-op). Do not copy the
legacy allow-all anon table policies for a production dataset.

## Notes

- Price history starts accumulating from the first time you visit a product
- More data gives the heuristic stronger context (`TRACKING` transitions to a
  history-based state or `noPatternMatch` at 7+ price observations)
- The extension only tracks products you actually visit (not all products on the site)
- Product identification uses site-specific URL-derived keys; EAN/GTIN is
  captured separately when available
- All prices are displayed in EUR regardless of language setting

## Not implemented yet

- Cloud history is not read back into verdicts or charts.
- EAN/GTIN values are searchable and copyable, but products are not yet
  matched or compared across stores.
- Price targets are checked when a product is visited; there are no proactive
  background or operating-system alerts.
- The tracked repository has no automated test runner. Saved-page replay
  tools and HTML fixtures are local development material and are gitignored.

## For developers / AI agents

See [CLAUDE.md](CLAUDE.md) for general behavioral guidelines plus this project's
specific rules (version-bump policy, conventions, where things live).

## Packaging for distribution

When zipping the extension for the Chrome Web Store or sideloading, exclude
these files so they don't ship to users:

- `test/` — manual test suite, not used at runtime
- `HTML pages and links/` — saved reference HTML samples used while writing
  the content scripts (32 MB of dev-only material, gitignored)
- `Emag.bg html.txt`, `Ozone.bg html.txt` — early saved-page reference dumps
- `GPT 5.5 audit.md`, `composer 2.5 multi agent audit.md`, `AGENTS.md` — internal AI-agent / audit notes
- `promo-small-440x280.png` — Chrome Web Store promo asset, uploaded
  separately via the Developer Dashboard (not part of the extension)
- `CLAUDE.md` — internal AI-agent rules
- `PRIVACY.md` — keep in the repo for the Web Store listing link, but the
  zip itself doesn't need to ship it
- `README.md` — optional, the Web Store listing already describes the extension

The required files in the .zip are: `manifest.json`, `background/`,
`content/`, `popup/`, `ui/`, `utils/`, `i18n/`, and `icons/`.

### Chrome Web Store assets (uploaded via Developer Dashboard, not bundled)

- **Store icon** `128×128` — taken from `icons/icon128.png`
- **Small promo tile** `440×280` — `promo-small-440x280.png`
- Optional larger promo / marquee tiles can be added later

These are configured under **Store listing → Graphic assets** in the
Chrome Web Store Developer Dashboard.

## License

MIT — see [LICENSE](LICENSE).

## Privacy

This extension stores your local price history in your browser, AND
attempts a best-effort upload of each recorded daily snapshot (product URL, title, EUR price, EAN,
observation date, plus a random pseudonymous device ID) to a Postgres
database hosted on Supabase. The current database-access issue and exact
field list are disclosed in [PRIVACY.md](PRIVACY.md).

No personally identifiable information, payment data, authentication
state, or browsing history outside the 20 supported stores is collected.

See [PRIVACY.md](PRIVACY.md) for the full disclosure, the complete
list of uploaded fields, and how to request data removal.
