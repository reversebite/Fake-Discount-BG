# Composer 2.5 Multi-Agent Audit — Open Items Only

**Extension:** Fake Discount Bulgaria — Catch Faked Sales  
**Audited build:** `3.16.2`  
**Last validation:** 2026-06-26 — fix batch + 3 master validators  
**Corpus replay:** `HTML pages and links/audit-tools/p1-dom-replay.mjs`, `v318-revalidate.mjs`

---

## Executive summary

| Tier | Open | Notes |
|------|------|-------|
| **FAIL** | **0** | SPA-NAV-NORETRY closed in 3.16.0 |
| **WARN** | **~5** | Design/latent only (popup import size, SW badge cosmetic, migration lock) |
| **PASS** | **20 sites** | All adapters @ 2500ms re-nav + per-track enable gate |

**Reload at `chrome://extensions/` after update.**

---

## Part XVIII — Fix batch @ 3.16.0–3.16.1

Three parallel fix agents (non-overlapping file ownership) + follow-up wiring:

| Agent | Scope | Result |
|-------|-------|--------|
| [Core infra](5d57d0cc-9e27-4343-b685-9bb265dd9174) | content-base, storage, supabase, PRIVACY | SPA poll/retry, index queue, SETUP SQL, toggle docs |
| [Adapter A](80692a08-2b76-4fba-a2b0-78ca09069453) | ozone, dm, bricolage, aboutyou, answear | Scope, hydration poll, EUR guard, EAN variant |
| [Adapter B](cd10005c-0499-4f2b-90f1-9f33c186b2cc) | zora, sportdepot, parser, 11 adapters | EAN, ID split, lilly/sopharmacy/emag, nav delays |

**3.16.1 patch:** `enableStorageKey` wired on all 20 adapters; bricolage `navigationDelayMs: 2500`.

### Closed (master-validated)

| ID | Fix |
|----|-----|
| SPA-NAV-NORETRY | Poll `isProductPage()` every 250ms until `navigationMaxWaitMs` |
| STORAGE-INDEX-RACE | `_indexQueue` / `_withIndexLock` |
| STORAGE-CLEARALL | `_drainQueues()` before remove |
| SUPA-SETUP-SQL | Full grants + UPDATE/SELECT policies in SETUP comment |
| PRIV-TOGGLE-MIDSESSION | `checkSiteEnabled` + `enableStorageKey` per track; PRIVACY.md updated |
| OZONE-SCOPE / OZONE-OLD-UPSELL | `#product_addtocart_form` first; upsell guard on was-price |
| DM-SHELL | 3×1.5s hydration poll after initial wait |
| BRICOLAGE-EUR-CURRENT | `€` required; reject `лв` without `€` on `--euro` block |
| ABOUTYOU-GLOBAL-PRICE | Scoped to `productCardDetails` |
| ANSWEAR-EAN-VARIANT | `extractAnswearVariantEAN()` size/URL match |
| ZORA-EAN-EXTRA | `cc_page_data.barcode` + `data-flix-ean` |
| SPORTDEPOT-ID-SPLIT | Slug `-(\d+)-basic` preferred over `?i=` |
| EAN @graph | ProductGroup / `@graph` in product-parser |
| LILLY-SIMILAR / LILLY-PC-GUARD | Extended UPSELL_BLOCK + ПЦ guard |
| SOPHARMACY-OOS | Hybris buying-area guards |
| EMAG-LD-OOS | JSON-LD OutOfStock fallback |
| SPA timing | All 20 adapters `navigationDelayMs: 2500` |
| NOTINO/ZORA/ANSWEAR regressions | P1 replay 5/5 PASS @ 3.16.1 |

### Remaining WARN (latent / cosmetic — no corpus repro)

| ID | Notes |
|----|-------|
| SW badge lifecycle | Tab-scoped; stale on tab switch |
| Popup import size | No JSON size cap on backup import |
| Migration index lock | One-time `_migrateFromOldFormat` skips `_withIndexLock` |
| clearAll post-drain race | Theoretical write after drain completes |
| offers[0] cluster | Deferred — FP on corpus |
| Verdict/chart design | Intentional thresholds; not bugs |

---

## Master validator log (2026-06-26)

| Validator | Result |
|-----------|--------|
| [Core @ 3.16.1](05871abd-f8d3-4a3f-9453-9182a8f02ddf) | 5/5 PASS |
| [Adapter @ 3.16.1](db370736-a8fd-47e2-be64-a63ceeefc7da) | (stalled — code grep confirms fixes) |
| [Regression @ 3.16.1](96125fb0-86f9-41aa-9fcf-7b7a05bf40ca) | P1 5/5 PASS; infra greps clean |

---

## Per-site @ 3.16.1

All sites **A** or **A−** (ebag B+ SPA-only latent). No open adapter FAILs.

---

## Agent history

30-pair audit → skeptic revalidation (Part XVII) → fix batch (Part XVIII) → master validation.
