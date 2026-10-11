---
name: tiktok-pixel-gtm-zid
description: Use when asked to install, set up or audit the TikTok Pixel or TikTok conversion events through Google Tag Manager on a Zid store, using a GTM MCP server.
---

# TikTok Pixel via GTM — Zid Stores

## Overview

Zid pushes a known, GA4-shaped dataLayer, so this build is fixed: one base tag plus five event tags on TikTok's own gallery template (`tiktok/gtm-template-pixel`), driven by Zid's events. Nothing about platform, events or field mapping needs asking — only three inputs.

The Zid dataLayer shape, every GTM MCP call and payload, and the post-publish checklist live in `reference.md`. Read it before Step 2.

## Step 0 — Inputs gate (do not start without all three)

```
GTM account ID or name .... <e.g. 1234567890  or  Client Name>
GTM container ID .......... <e.g. GTM-XXXXXXX>
TikTok Pixel ID ........... <20 characters, e.g. C4XXXXXXXXXXXXXXXXXX>
```

If any is missing: ask for it and **stop**. **Never build with a placeholder pixel ID.** If the real ID isn't available yet, build everything but leave every TikTok tag **paused** and say so — a published base tag with a fake ID runs `ttq.load('<garbage>')` on every pageview and logs "Invalid pixel ID".

## What gets built

| Tag | TikTok event | Fires on (Zid dataLayer event) |
|---|---|---|
| `EA - TikTok Pixel - Base Code` | *(loads `ttq`, fires Pageview)* | All Pages (`2147479553`), **priority 100** |
| `EA - TikTok Pixel - ViewContent` | ViewContent | `view_item` |
| `EA - TikTok Pixel - AddToCart` | AddToCart | `add_to_cart` |
| `EA - TikTok Pixel - InitiateCheckout` | InitiateCheckout | `begin_checkout` |
| `EA - TikTok Pixel - AddPaymentInfo` | AddPaymentInfo | `add_payment_info` |
| `EA - TikTok Pixel - CompletePayment` | CompletePayment *(see purchase event name below)* | `purchase` |

Plus folders `EA - TikTok Pixel` and `EA - Core`, the constant `EA - TikTok Pixel - pixel_id`, two Custom JS variables, and only the triggers that don't already exist.

> **The TikTok template does not load the pixel.** Its `validate()` fails with `"ttq not found"` unless `ttq` already exists. The Custom HTML base tag is mandatory.

Fixed decisions — do not re-litigate:
- Manual contents: `enhance_ecomm: false` and `single_multi_product: "multiple"`. Not TikTok's "Use Google Analytics E-commerce Data Layers" toggle.
- Advanced Matching in **hashed** mode, using Zid's pre-hashed email, phone and external ID.
- `value` is always `ecommerce.value`, never recomputed from items.
- `order_id` and `event_id` on the purchase tag = `ecommerce.transaction_id`.

### Purchase event name

TikTok renamed `CompletePayment` to **`Purchase`** for web on 1 May 2025. The legacy name is converted automatically and supported indefinitely. Read the installed template's `event` dropdown in Step 2: if it offers `Purchase`, use it and name the tag `EA - TikTok Pixel - Purchase`; if it only offers `CompletePayment`, build it as written in `reference.md`. Never type an event the dropdown doesn't contain — the template logs a warning for unknown events.

## Workflow

Follow `reference.md` → **Build procedure**, Steps 1–10, in order. The additions below come from live builds and are not optional.

1. **Locate the container** (Step 1). If a later write returns *"Workspace is already submitted"*, the client published in the meantime — list workspaces again and continue in the new one.
2. **Get the template's tag type** (Step 2) from `___INFO___`, and read the `event` dropdown for the purchase event name. Never guess either.
3. **Audit** (Step 3). List calls return 20 items per page — read `totalPages` and fetch every page before deciding something doesn't exist.
4. **Check for an existing TikTok install** (Step 4). If a browser tool is available, also run the live-site check below.
5. **Folders → variables → triggers → tags** (Steps 5–8). Base tag first. Set `parentFolderId` at creation time.
6. **Verify** (Step 9), **report and stop** (Step 10). Never publish.

### Live-site check (when a browser tool is available)

On the store's homepage and a product page:
- Read the GTM script `src`. Zid stores have been observed loading `gtm.js?id=GTM-XXXX&l=gtmDataLayer` — a **custom dataLayer name**. `window.dataLayer` then exists but stays empty; Zid's events are in `window.gtmDataLayer`, the same array GTM listens to, so triggers work. Inspect the array named in `l=`, not `dataLayer`.
- Check whether `window.ttq` exists before GTM fires, or an `analytics.tiktok.com` script that GTM did not load. Two installs double-count every event.

## Hard rules

1. **Never publish and never create a container version.** Build, verify, report, stop.
2. **Never create a trigger that already exists** — reuse any Custom Event trigger matching the dataLayer event, whatever it is called.
3. **Never rename a pre-existing tag, trigger or variable.** GTM resolves variables by name; a rename silently breaks every `{{...}}` that points at it.
4. **Naming:** tags `EA - TikTok Pixel - <TikTokEventName>`; TikTok variables `EA - TikTok Pixel - <snake_case>`; shared variables `EA - Core - <snake_case>`.
5. **The pixel ID lives only in `EA - TikTok Pixel - pixel_id`** — including inside the base Custom HTML.
6. Every entity gets a folder and a `notes` value. Shared customer variables go in `EA - Core`.

## Common mistakes

| Mistake | What actually happens |
|---|---|
| No base Custom HTML tag | Every event tag fails with `"ttq not found"` and sends nothing. |
| Plain-mode Advanced Matching | The `hash` radio covers email, phone **and** external ID together. In plain mode the template hashes all three, but Zid's `customer.external_id` is already SHA-256 — that match key is destroyed. The template's own `validate()` warns `"data.email appears to be a SHA256 hash"` in the reverse case. |
| Turning on the GA4 auto-ecommerce toggle | The template then computes `value` as Σ(price × quantity). On Zid's `purchase` that is far below `ecommerce.value`, so TikTok reports different revenue from the other platforms. |
| Tag `type` guessed as `cvt_<containerId>_<templateId>` | `400 vendorTemplate.key: Unknown entity type`. The type is the template's own ID from `___INFO___` (currently `cvt_MRQN8`). |
| Publishing with a placeholder pixel ID | `ttq.load` runs with a fake ID on every pageview; nothing is collected. Pause the tags instead. |
| `update` with a partial object | `update` replaces the entity. Omitted `parentFolderId`, `firingTriggerId`, `priority` and `notes` are deleted. Send the full object plus the current `fingerprint`. |
| Treating a `429 Quota exceeded` as a bad payload | It is the per-minute API limit. Wait and retry. |

## Field notes from live Zid builds

- **Guests carry no customer data.** For a logged-out visitor Zid sends `customer: { is_guest: true }` and nothing else, so Advanced Matching is empty on guest product views. Expected.
- **The base snippet in `reference.md` is TikTok's standard published code**, not pulled from the client's account. Ask for it to be compared with TikTok Events Manager → Install Pixel → Install Code Manually before publishing.
- **`contents` must return `undefined`, not `[]`, when empty.** An empty array tells TikTok the cart is empty, which is a different claim from "no data yet".
- **COD stores.** `payment_type: "zid_cod"` means `purchase` fires at order placement, not settlement; undelivered orders inflate conversions. TikTok's `PlaceAnOrder` describes that moment better but is a legacy event retiring in 2027 — don't build it; say so in the report.
- **Verifying in a browser:** pixel requests are often POST/`fetch`, which Resource Timing does not expose. Use the TikTok Pixel Helper or hook `fetch`/`sendBeacon`.

## Events Zid fires that this skill does not build

`view_cart`, `add_shipping_info`, `remove_from_cart` — no TikTok standard event. `add_to_wishlist` — Zid sends it with the item and its `item_category`, and TikTok has a standard `AddToWishlist`; build it only if asked, on the same pattern as AddToCart.

## Report

State: account / container / workspace / folder IDs and the template ID used; which purchase event name was used and why; what was created versus reused; whether an existing TikTok install was found; the live-site check results; that the base snippet needs comparing with Events Manager; and whether tags were left paused for a missing pixel ID. Then stop.
