# Privacy Policy — Fake Discount Bulgaria

**Last updated:** 15 July 2026

## Summary

Fake Discount Bulgaria is a browser extension that shows history-based price
signals on 20 Bulgarian e-commerce sites by recording the prices of products you
visit. Local history remains the source used by the extension, and **a
pseudonymous copy of each recorded daily product observation is uploaded to a developer-controlled
Postgres database (Supabase)** so the extension can build a shared
price-history dataset across all installs.

This page describes exactly what is stored locally, what is uploaded,
and what is **not** collected.

## Supported sites

The extension only runs on these 20 Bulgarian e-commerce domains:

`Emag.bg`, `Ozone.bg`, `Notino.bg`, `Technopolis.bg`, `Technomarket.bg`,
`Zora.bg`, `Ardes.bg`, `Plesio.bg`, `Aboutyou.bg`, `Answear.bg`,
`Decathlon.bg`, `dm-drogeriemarkt.bg`, `Fashiondays.bg`,
`Lillydrogerie.bg`, `Mr-bricolage.bg`, `Obuvki.bg`, `Praktiker.bg`,
`Sopharmacy.bg`, `Sportdepot.bg`, `eBag.bg`.

It does not run on any other site and does not read any other site's page
content. Because the extension uses Chrome's `tabs` permission for
single-page-app navigation refreshes, the service worker may receive URL
change events for open tabs, but it does not store or upload URLs outside
the supported domains.

## What is stored locally (in your browser)

When you visit a product page on one of the supported sites, the
extension stores the following in your browser's local extension storage
(`chrome.storage.local`). This data stays on your computer:

- The product URL and an extension-internal product ID
- The product title
- The thumbnail image URL
- The product's EAN/GTIN barcode when the page exposes one
- The current price (and the seller's claimed "original price", if shown)
- The date of each visit (one observation per day, per product)
- Optional: a price target you set manually
- Your settings (language, per-site toggles, chart visibility,
  popup filter chips, sort preference)
- A random pseudonymous device ID (UUID v4) used to deduplicate uploads
  from the same installation

Hiding the on-page chart (Settings → "Show chart on product pages") does
**not** stop local price tracking or Supabase uploads — it only controls
whether the widget is drawn on the product page.

Turning off **Track prices on** a specific store stops tracking, widget
injection, and cloud uploads for that store only. Other enabled stores
continue to record and upload as usual. The change takes effect on the
**next page load or in-page navigation** for that store (Manifest V3
content scripts are injected once per document lifecycle), not instantly
on tabs that are already open when you flip the toggle.

This local data is used to build a price history for each product, to
calculate heuristic deal signals, and to render the
price-graph widget on product pages.

## What is uploaded to Supabase

In addition to the local copy, the extension attempts a best-effort upload of
each recorded daily price snapshot to a Postgres database hosted on Supabase,
operated by the developer of this extension. The upload is fire-and-forget,
has no durable retry queue, and never blocks the local save or widget render.
Offline or interrupted uploads can therefore be missing remotely. Each upload contains:

| Field | Example | Purpose |
|---|---|---|
| `device_id` | random UUID v4 | Deduplicate observations from the same install. Not linked to your identity. |
| `product_id` | `emag_DKFWLW3BM` | Extension-internal identifier derived from the URL. |
| `site` | `emag` | Which store the observation comes from. |
| `url` | `https://www.emag.bg/...` | The full product page URL as visited, including query parameters or a fragment when present. |
| `title` | `Smartphone Samsung Galaxy S25 FE` | The product name. |
| `thumbnail` | `https://cdn.emag.bg/...jpg` | The product image URL. |
| `ean` | `8806097540519` | The EAN/GTIN barcode when the page exposes one. |
| `price` | `499.00` | Current displayed EUR price. |
| `original_price` | `599.00` | Seller's claimed "was" price, when shown. |
| `discount` | `17` | Percentage difference between the two. |
| `observed_date` | `2026-05-14` | Date of the observation (local time). |
| `observed_at` | server timestamp | Supabase/Postgres timestamp for when the upload was received. |
| `ext_version` | `3.16.4` | Extension version that recorded the observation. |
| `user_agent` | full browser UA string | Browser/OS identification, for debugging extraction issues. |

The upload is keyed by `(device_id, product_id, observed_date)` — only
one record per product per device per day is kept, so repeated visits
within the same day don't bloat the dataset.

## How the uploaded data is used

The dataset is used by the developer to:

- Verify that the extension's verdict logic produces consistent results
  across the user base.
- Detect store-side changes (new HTML layouts, removed price markup,
  currency mix-ups) that break extraction on individual installs.
- Aggregate observed prices for future features such as cross-install
  price-history sharing or community-wide fake-discount detection.

The developer does not sell the dataset, provide it to advertising networks,
or use it for targeted advertising. The production Supabase endpoint currently
allows a query made with the public anon key bundled in the extension to read
rows from `price_history`. Uploaded rows must therefore not be treated as
private to the developer. The documented database recipe also defines an
allow-all anon update policy but omits the table grant needed for the extension
upsert: as written, fresh writes fail; adding the missing grant would let any
anon-key holder update arbitrary rows. Direct table access must be replaced by
a controlled write path before public release.

Uploaded rows currently have no automatic expiration and are retained until
the developer removes them or the database policy changes.

There is **no in-extension toggle** to disable cloud uploads in the
public build. Forks can blank the Supabase constants in
`utils/supabase-sync.js` for a local-only build.

## What the extension does NOT collect

- **Personally identifiable information** — no name, email, address,
  phone number, account ID, or social media handle is written by the
  extension. Supabase receives normal request metadata such as IP address
  as part of hosting the upload endpoint, but the extension does not store
  IP addresses in the `price_history` table.
- **Payment or financial information** — no card numbers, bank details,
  invoices, or transaction records.
- **Authentication data** — no passwords, cookies, session tokens, or
  login state.
- **Browsing history outside the supported sites** — product observations
  are recorded and uploaded only for supported store pages. URL-change
  events outside the supported domains may be visible to the service worker
  through the `tabs` permission, but they are discarded and never stored or
  uploaded.
- **Page content beyond product details** — no reviews, comments,
  account dashboards, cart contents, or order history.
- **Advertising trackers or active fingerprinting** — there is no analytics
  SDK or third-party advertising tag. However, the same random pseudonymous
  device ID is used for observations across all enabled supported stores, so
  those retailer visits are linkable to one extension installation. Uploaded
  rows also include the full user-agent string listed above.

## How to delete your data

**Local data:**

- Open the popup → **Data** tab → **"Clear all history"** to remove indexed
  product records. A known race with simultaneous page tracking can leave an
  orphaned storage key; close supported product tabs and repeat if storage
  usage does not fall as expected.
- **"Cleanup old"** removes whole products not seen for at least 90 days.
- Uninstalling the extension from `chrome://extensions/` removes all
  local data.

**Uploaded data:**

Because the upload is keyed only by a random `device_id` that the
extension generates locally, the developer cannot identify which rows
belong to you without you sending the device ID first. To request
removal of uploaded observations from your install, email the contact
address below and include your device ID (find it in your browser's
DevTools console under `chrome.storage.local` → `supabase_device_id`).
Concurrent first-run uploads can currently generate a transient second device
ID before one is persisted. Rows written under that lost ID cannot be located
from the retained ID; this must be fixed before public release.

## Export and import

The extension lets you export locally stored price-history product records as
a JSON file (a manual local backup) and import them later. Settings, price
targets, popup filters, and the Supabase device ID are not included. These backup files are
created and read only on your computer, by your own action; they are
never uploaded anywhere by the extension.

On import, the extension validates that every product URL belongs to one of
the 20 supported store domains and accepts thumbnail URLs only when they are
syntactically valid `https://` URLs. Any HTTPS host is allowed, and the popup
will request that image when rendering the imported product, so only trusted
backup files should be imported. Products with invalid store URLs or without
any valid history row are skipped; a non-HTTPS thumbnail is removed while the
rest of an otherwise valid product is kept. The importer has no file-size cap,
is not transactional, and can partially replace existing records before a
later storage error is reported.

## Permissions explained

- `storage` — to save the price history locally
- `tabs` — to detect when you navigate between product pages on
  single-page-app sites (so the chart updates without a hard reload)
- Host access to the 20 supported store domains — to read product
  prices from those pages and inject the price-history chart
- Host access to `gdfsqujcjqktjhhgkxbs.supabase.co` — to upload each
  price observation to the developer's database

## Changes to this policy

If the extension changes what is collected, what is uploaded, or how
either is used, this policy will be updated and the new version will be
linked from the Chrome Web Store listing before the change ships.

## Contact

Questions, complaints, or data-removal requests:
**fakediscountbg@gmail.com**
