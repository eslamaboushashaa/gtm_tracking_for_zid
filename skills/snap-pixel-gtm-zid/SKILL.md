---
name: snap-pixel-gtm-zid
description: Use when asked to install, set up or audit the Snapchat (Snap) Pixel or Snap conversion events through Google Tag Manager on a Zid store, using a GTM MCP server.
---

# Snap Pixel via GTM — Zid Stores

## Overview

Zid pushes a known, GA4-shaped dataLayer, so this build is fixed: six Snap Pixel tags on Snapchat's "Snap Pixel" gallery template, driven by Zid's own events. Nothing about platform, events or field mapping needs asking — only three inputs.

The Zid dataLayer shape, every GTM MCP call and payload, and the post-publish checklist live in `reference.md`. Read it before Step 2.

## Step 0 — Inputs gate (do not start without all three)

```
GTM account ID or name .... <e.g. 1234567890  or  Client Name>
GTM container ID .......... <e.g. GTM-XXXXXXX>
Snap Pixel ID ............. <UUID, e.g. 12345678-abcd-1234-abcd-1234567890ab>
```

If any is missing: ask for it and **stop**. No GTM reads, no browsing.

## What gets built

| Tag | Snap `event_type` | Fires on (Zid dataLayer event) |
|---|---|---|
| `EA - Snap Pixel - PAGE_VIEW` | `PAGE_VIEW` | All Pages (built-in trigger `2147479553`) |
| `EA - Snap Pixel - VIEW_CONTENT` | `VIEW_CONTENT` | `view_item` |
| `EA - Snap Pixel - ADD_CART` | `ADD_CART` | `add_to_cart` |
| `EA - Snap Pixel - START_CHECKOUT` | `START_CHECKOUT` | `begin_checkout` |
| `EA - Snap Pixel - ADD_BILLING` | `ADD_BILLING` | `add_payment_info` |
| `EA - Snap Pixel - PURCHASE` | `PURCHASE` | `purchase` |

Plus folders `EA - Snap Pixel` and `EA - Core`, the constant `EA - Snap Pixel - pixel_id`, six Custom JS variables, and only the triggers that don't already exist.

**The Snap template loads the pixel itself** (it injects `scevent.min.js` and calls `snaptr('init')`), so there is no separate base tag — the PAGE_VIEW tag does the init.

Fixed decisions — do not re-litigate:
- `item_ids` is a real **array**, never a comma-joined string.
- Advanced Matching uses Zid's **pre-hashed** values through `user_hashed_email` / `user_hashed_phone_number`.
- `price` is always `ecommerce.value`, never recomputed from items.
- `gateway_url` is never set.

## Workflow

Follow `reference.md` → **Build procedure**, Steps 1–10, in order. The additions below come from live builds and are not optional.

1. **Locate the container** (Step 1). If a later write returns *"Workspace is already submitted"*, the client published in the meantime — list workspaces again and continue in the new one.
2. **Get the template's tag type** (Step 2). Never guess it; read it from `___INFO___`. If the template isn't installed, stop and ask for it to be added from the Community Template Gallery ("Snap Pixel" by Snap Inc.).
3. **Audit** (Step 3). List calls return 20 items per page — read `totalPages` and fetch every page before deciding something doesn't exist.
4. **Check for an existing Snap install** (Step 4). If a browser tool is available, also run the live-site check below.
5. **Folders → variables → triggers → tags** (Steps 5–8). Set `parentFolderId` at creation time, on variables as well as tags.
6. **Verify** (Step 9), **report and stop** (Step 10). Never publish.

### Live-site check (when a browser tool is available)

On the store's homepage and a product page:
- Read the GTM script `src`. Zid stores have been observed loading `gtm.js?id=GTM-XXXX&l=gtmDataLayer` — a **custom dataLayer name**. `window.dataLayer` then exists but stays empty; Zid's events are in `window.gtmDataLayer`, the same array GTM listens to, so triggers work. Inspect the array named in `l=`, not `dataLayer`.
- Check for an `sc-static.net` / `tr.snapchat.com` script that GTM did not load, or a `snaptr` global before GTM fires. Two installs double-count every event.

## Hard rules

1. **Never publish and never create a container version.** Build, verify, report, stop.
2. **Never create a trigger that already exists** — reuse any Custom Event trigger matching the dataLayer event, whatever it is called.
3. **Never rename a pre-existing tag, trigger or variable.** GTM resolves variables by name; a rename silently breaks every `{{...}}` that points at it.
4. **Naming:** tags `EA - Snap Pixel - <SNAP_EVENT_TYPE>`; Snap variables `EA - Snap Pixel - <snake_case>`; shared variables `EA - Core - <snake_case>`.
5. **The pixel ID lives only in `EA - Snap Pixel - pixel_id`.** Never hardcode it in a tag.
6. Every entity gets a folder and a `notes` value. Shared customer variables go in `EA - Core`, not the Snap folder.

## Common mistakes

| Mistake | What actually happens |
|---|---|
| Tag `type` guessed as `cvt_<containerId>_<templateId>` | `400 vendorTemplate.key: Unknown entity type`. The type is the template's own ID from `___INFO___` (currently `cvt_K4VXG`). |
| Setting `gateway_url` | It is the "CAPI Gateway Script URL" with a built-in default. Setting it pins the script URL and implies a Conversions API Gateway that doesn't exist. The UI repopulates the default when cleared, so it only *looks* set. |
| Looking for an `external_id` field | Snap has none. Zid's `customer.external_id` is usable on Meta and TikTok only. |
| Adding `uuid_c1` / `_scid` to the tags | It is a Conversions API (server-side) parameter. The browser pixel writes the `_scid` cookie itself and the template has no field for it. |
| "Harmonising" PII with the Meta build | Snap takes Zid's pre-hashed values; Meta takes the plain ones. That difference is deliberate. |
| Sending `item_ids` as a comma string | Snap's documented format is an array. The Custom JS variable already returns one. |
| `update` with a partial object | `update` replaces the entity. Omitted `parentFolderId`, `firingTriggerId` and `notes` are deleted. Send the full object plus the current `fingerprint`. |
| Treating a `429 Quota exceeded` as a bad payload | It is the per-minute API limit. Wait and retry. |

## Field notes from live Zid builds

- **Guests carry no customer data.** For a logged-out visitor Zid sends `customer: { is_guest: true }` and nothing else, so Advanced Matching is empty on guest product views. Expected.
- **`item_category` exists on `view_item` only, and `item_description` on `purchase` only.** If those fields are wired onto other events they simply resolve to nothing there.
- **Variables created without `parentFolderId` don't show in the folder.** Fix with `gtm_folder action=moveEntitiesToFolder` and the `variableId` array.
- **COD stores.** `payment_type: "zid_cod"` means `purchase` fires at order placement, not settlement; undelivered orders inflate conversions. Say so in the report.
- **Verifying in a browser:** pixel requests are often POST/`fetch`, which Resource Timing does not expose. Use the Snap Pixel Helper or hook `fetch`/`sendBeacon`.

## Events Zid fires that this skill does not build

`view_cart`, `add_shipping_info`, `remove_from_cart` — no Snap standard event. `add_to_wishlist` — Zid sends it with the item and its `item_category`, and Snap has a standard `ADD_TO_WISHLIST`; build it only if asked, on the same pattern as ADD_CART.

## Report

State: account / container / workspace / folder IDs and the template ID used; what was created versus reused; whether an existing Snap install was found; the live-site check results; and that `gateway_url` was left untouched. Then stop.
