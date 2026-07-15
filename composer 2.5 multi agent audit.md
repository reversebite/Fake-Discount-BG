# Fake Discount Bulgaria Audit Status

**Runtime audited build:** `3.16.3` (`4c7cc19`)
**Documentation correction:** `3.16.4`
**Follow-up audit:** 15 July 2026
**Status:** **NOT READY FOR PUBLIC RELEASE**

This document supersedes the earlier Composer 2.5 conclusion that all 20
adapters passed with no open failures. That conclusion was too strong: one
validator stalled, the replay tools and HTML fixtures are gitignored rather
than reproducible from the repository, and some validators treated a missing
price as a pass.

## Release blockers

| Severity | Finding | Evidence |
|---|---|---|
| **P0** | The configured Supabase `anon` role can read the shared `price_history` table. The public key is necessarily bundled with the extension, so uploaded rows are not private. The documented recipe also defines allow-all `UPDATE` but omits the anon table grant: fresh writes fail as written, while adding the grant makes arbitrary row mutation possible. | `utils/supabase-sync.js:31-43`; read-only production probe returned HTTP `200` with a non-empty one-row response. No row contents were retained or displayed. |
| **P1** | Ozone can record the wrong current price from a correctly scoped main-product node. The Reanimal fixture contains `29,99 EUR / 58,66 BGN`; passing the whole node to `parsePrice()` returns `30.00` instead of `29.99`. | `content/ozone.js:124-133`; actual-adapter replay failed 1 of 4 Ozone fixtures. |
| **P1** | SportDepot product IDs are unstable. `HM6803-101-basic` becomes `sportdepot_101`, while another saved sample falls back to its whole slug. Common color suffixes can collide across unrelated products and variant URLs can split history. | `content/product-parser.js:81-96`; saved SportDepot fixtures expose stable JSON-LD `mpn` values such as `NIKE-HM6803`. |
| **P1** | Concurrent migration or Clear all can orphan product keys outside `product_index`; they disappear from popup/export/count while still consuming storage. Both races were reproduced with the committed manager. | `utils/storage.js:55-66`, `331-340`. |
| **P1** | Import commits sequential overwrites without preflight or rollback. A later quota/storage failure leaves earlier products replaced while the popup reports a generic failure. | `background/service-worker.js:319-329`; `utils/storage.js:274-286`. |
| **P1** | Concurrent first uploads can return two different device IDs while only one remains stored. Remote rows under the lost ID cannot be found through the documented deletion process. | `utils/supabase-sync.js:87-97`; reproduced with two concurrent calls. |
| **P1** | `FAKE_DISCOUNT` can be emitted when no promotion or claimed original price exists: current price only needs to be more than 10% above the observed 30-day low. This is a heuristic price-warning signal, not proof that a retailer faked a sale. | `background/price-tracker.js:109-128`. |

## Other open risks

| Severity | Finding |
|---|---|
| **P2** | Imported product IDs are not constrained or matched to the URL/site, history is not sorted, and there is no input-size cap. Import can overwrite an existing product record (`background/service-worker.js:304-414`). |
| **P2** | `ProductParser.parsePrice()` reads US-style `1,199.00 EUR` as `1.20` (`content/product-parser.js:178-188`). Bulgarian saved samples primarily use comma-decimal formats, so this is latent rather than a reproduced site failure. |
| **P2** | Cloud sync has no durable outbox or retry. Offline/terminated uploads are silently absent from the pooled dataset even though local history succeeds (`background/service-worker.js:469-488`; `utils/supabase-sync.js:108-158`). |
| **P2** | Extraction failure and out-of-stock both render the same zero-history `TRACKING` state. This masks adapter regressions and allowed the old Obuvki simulator to report pass with `price:null`; actual adapter replay did extract those visible prices (`content/content-base.js:125-156`). |
| **P2** | Badge state is not explicitly cleared when a tab leaves a product page; stale per-tab verdict/target badges remain possible (`background/service-worker.js:56-61`, `498-529`). |
| **P2** | Concurrent tabs update one monolithic `priceTargets` object with unlocked read-modify-write operations, so simultaneous edits can lose a target (`ui/price-graph-widget.js:417-460`). |
| **P2** | Imported HTTPS thumbnails may point to any host and are requested by the popup, allowing a crafted backup to trigger a remote tracking request (`background/service-worker.js:355-366`; `popup/popup.js:494`). |
| **P2** | There is no tracked automated test runner. `test/`, saved HTML, and replay tools are gitignored; the old browser-console test is local, stale, and cannot exercise popup-only and content-only message actions in one context. |
| **P3** | Technomarket fixtures expose valid EANs in `<tm-pointandplace ean>`, but the generic extractor does not inspect that attribute (`content/technomarket.js:152`; `content/product-parser.js:265-293`). |

## Portfolio claim corrections

- **20-site coverage:** 20 configured adapters are verified. This is not the
  same as a current live pass on all 20 retailers.
- **Automatic tracking:** visit-triggered daily snapshots on enabled,
  recognized product pages with a valid purchasable price; no continuous
  monitoring.
- **Five verdicts:** implemented, but they are heuristic price states.
- **Cloud sync:** best-effort one-way ingestion. The extension does not read
  pooled Supabase history back into charts or verdicts.
- **EAN cross-site matching:** EAN/GTIN capture, validation, search, and copy
  are implemented. Cross-store matching/comparison is not.
- **Price-target alerts:** target lines, pills, popup markers, and a visit-time
  tab badge are implemented. Background polling and browser notifications are
  not.
- **SPA / anti-bot:** SPA navigation retries, hydration waits, scoped DOM
  extraction, and isolated-world CSP resilience are implemented. No anti-bot
  bypass exists.
- **Import/export:** sanitized price-history product records only, not a full
  settings/targets backup.

## Checks completed

- Pushed and audited commit `4c7cc19` on `main`.
- `node --check` passed for all 29 modified runtime JavaScript files.
- Manifest and both locale JSON files parse; BG/EN key parity is `141/141`.
- Manifest contains 20 content-script blocks, 40 host variants, and no missing
  host or web-accessible-resource match.
- All 20 adapters have a per-track enable gate and `navigationDelayMs: 2500`.
- DOM construction avoids product-data `innerHTML`; widget event isolation is
  bubble-phase only.
- Existing saved-page replay checks were run, but they are local, incomplete,
  and contain false-pass conditions. They are supporting evidence, not a
  release gate.
- Actual-adapter replay covered 68 saved pages: 67 usable products extracted
  without browser-code exceptions. Current/original prices matched on 19
  sites; Ozone passed 3/4 because of the dual-currency bug above. Twelve saved
  out-of-stock pages across nine stores correctly returned `price:null`.

## Recommended release order

1. Replace direct anonymous table access with controlled ingestion; revoke
   anon `SELECT` / `UPDATE`, then re-test uploads and update `PRIVACY.md`.
2. Fix Ozone dual-currency parsing and add an exact-value regression fixture.
3. Re-key SportDepot from a verified stable identifier and migrate or isolate
   existing `sportdepot_101` history.
4. Serialize migration, clear-all, device-ID creation, and target writes; make
   import preflighted/transactional or clearly merge-safe.
5. Decide whether the 30-day-low rule should say `FAKE_DISCOUNT` when no sale
   is claimed; align labels, manifest copy, README, and tests.
6. Add a tracked deterministic test harness for verdicts, storage races,
   import sanitization, and fixture replay across all adapters.
7. Run a live Chrome regression sweep on all 20 stores before Web Store upload.
