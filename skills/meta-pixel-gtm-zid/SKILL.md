---
name: meta-pixel-gtm-zid
description: Use when asked to install, set up or audit the Meta (Facebook) Pixel or Meta conversion events through Google Tag Manager on a Zid store, using a GTM MCP server.
---

# Meta Pixel via GTM — Zid Stores

## Overview

Zid pushes a known, GA4-shaped dataLayer, so this build is fixed: six Meta Pixel tags on the facebook "Meta Pixel" gallery template, driven by Zid's own events. Nothing about platform, events or field mapping needs asking — only three inputs.

The Zid dataLayer shape, every GTM MCP call and payload, and the post-publish checklist live in `reference.md`. Read it before Step 2.

## Step 0 — Inputs gate (do not start without all three)

```
GTM account ID or name .... <e.g. 1234567890  or  Client Name>
GTM container ID .......... <e.g. GTM-XXXXXXX>
Meta Pixel ID ............. <e.g. 123456789012345>
```

If any is missing: ask for it and **stop**. No GTM reads, no browsing.

## What gets built

| Tag | Meta event | Fires on (Zid dataLayer event) |
|---|---|---|
| `EA - Meta Pixel - PageView` | PageView | All Pages (built-in trigger `2147479553`) |
| `EA - Meta Pixel - ViewContent` | ViewContent | `view_item` |
| `EA - Meta Pixel - AddToCart` | AddToCart | `add_to_cart` |
| `EA - Meta Pixel - InitiateCheckout` | InitiateCheckout | `begin_checkout` |
| `EA - Meta Pixel - AddPaymentInfo` | AddPaymentInfo | `add_payment_info` |
| `EA - Meta Pixel - Purchase` | Purchase | `purchase` |

Plus folders `EA - Meta Pixel` and `EA - Core`, the constant `EA - Meta Pixel - pixel_id`, three shared customer variables, five Custom JS variables, and only the triggers that don't already exist.

Fixed decisions — do not re-litigate:
- `content_ids` / `contents[].id` come from `item_id` (fallback `id`). Never the SKU field — Zid calls it `item_sku` on most events and `sku` on `purchase`.
- `value` is always `ecommerce.value`, never recomputed from items (on `purchase` it includes tax, shipping and fees).
- Advanced Matching sends the **plain** `customer.email` and `customer.mobile`; `fbq` hashes them itself.
- `optInMetaCAPI` = `false` and `consent` = `true` on every tag.
- Purchase carries `eventId` = `ecommerce.transaction_id` for later Conversions API deduplication.

## Workflow

Follow `reference.md` → **Build procedure**, Steps 1–10, in order. The additions below come from live builds and are not optional.

1. **Locate the container** (Step 1). If a later write returns *"Workspace is already submitted"*, the client published in the meantime — list workspaces again and continue in the new one.
2. **Get the template's tag type** (Step 2). Never guess it; read it from `___INFO___`.
3. **Audit** (Step 3). List calls return 20 items per page — read `totalPages` and fetch every page before deciding something doesn't exist.
4. **Check for an existing pixel** (Step 4). If a browser tool is available, also run the live-site check below.
5. **Folders → variables → triggers → tags** (Steps 5–8). Set `parentFolderId` at creation time.
6. **Verify** (Step 9), **report and stop** (Step 10). Never publish.

### Live-site check (when a browser tool is available)

On the store's homepage and a product page:
- Read the GTM script `src`. Zid stores have been observed loading `gtm.js?id=GTM-XXXX&l=gtmDataLayer` — a **custom dataLayer name**. `window.dataLayer` then exists but stays empty; Zid's events are in `window.gtmDataLayer`, the same array GTM listens to, so triggers work. Inspect the array named in `l=`, not `dataLayer`.
- Check whether `fbq` exists before GTM fires (a theme- or Zid-injected pixel). Two installs double-count every event.

## Hard rules

1. **Never publish and never create a container version.** Build, verify, report, stop.
2. **Never create a trigger that already exists** — reuse any Custom Event trigger matching the dataLayer event, whatever it is called.
3. **Never rename a pre-existing tag, trigger or variable.** GTM resolves variables by name; a rename silently breaks every `{{...}}` that points at it.
4. **Naming:** tags `EA - Meta Pixel - <MetaEventName>`; Meta variables `EA - Meta Pixel - <snake_case>`; shared variables `EA - Core - <snake_case>`.
5. **The pixel ID lives only in `EA - Meta Pixel - pixel_id`.** Never hardcode it in a tag.
6. Every entity gets a folder and a `notes` value.

## Common mistakes

| Mistake | What actually happens |
|---|---|
| Tag `type` guessed as `cvt_<containerId>_<templateId>` | `400 vendorTemplate.key: Unknown entity type`. The type is the template's own ID from `___INFO___` (currently `cvt_5RM3Q`). |
| Sending `customer.email_hashed` to `em` | Not a documented path for `fbq('init')`; a normalisation mismatch fails silently. Send the plain value. |
| Leaving `optInMetaCAPI` at its default | It defaults to `true` and authorises Meta to set up a Conversions API integration on the client's behalf. |
| Recomputing `value` from items | On `purchase` Σ(price × quantity) is far below `ecommerce.value`; platforms then disagree on revenue. |
| `update` with a partial object | `update` replaces the entity. Omitted `parentFolderId`, `firingTriggerId` and `notes` are deleted. Send the full object plus the current `fingerprint`. |
| Reading `gtm_template list` into context | ~120k characters. It is written to a file — parse it with a script. |
| Treating a `429 Quota exceeded` as a bad payload | It is the per-minute API limit. Wait and retry. |

## Field notes from live Zid builds

- **Guests carry no customer data.** For a logged-out visitor Zid sends `customer: { is_guest: true }` and nothing else, so Advanced Matching is empty on guest product views. Expected — it populates for identified customers and at checkout.
- **"Duplicate Pixel ID" in the console is normal.** The template re-runs `fbq('init')` on each tag to refresh advanced matching. `fbq.getState().pixels` showing one entry means one pixel.
- **`content_ids` correctness is invisible.** A mismatch with the catalog's retailer IDs keeps Events Manager green while Advantage+ catalog ads fail. Always flag it for checking in Commerce Manager.
- **COD stores.** `payment_type: "zid_cod"` means `purchase` fires at order placement, not settlement; undelivered orders inflate conversions. Say so in the report.
- **Verifying in a browser:** pixel requests are often POST/`fetch`, which Resource Timing does not expose. Use the Meta Pixel Helper or hook `fetch`/`sendBeacon`.

## Events Zid fires that this skill does not build

`view_cart`, `add_shipping_info`, `remove_from_cart` — no Meta standard event. `add_to_wishlist` — Zid sends it with the item and its `item_category`, and Meta has a standard `AddToWishlist`; build it only if asked, on the same pattern as AddToCart.

## Report

State: account / container / workspace / folder IDs and the template ID used; what was created versus reused; whether an existing pixel was found; the live-site check results; and that `content_ids` still needs checking against the catalog. Then stop.
