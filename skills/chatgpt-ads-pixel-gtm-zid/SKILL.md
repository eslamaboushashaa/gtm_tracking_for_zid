---
name: chatgpt-ads-pixel-gtm-zid
description: Use when asked to install, set up or audit the ChatGPT Ads / OpenAI measurement pixel (oaiq) or ChatGPT conversion events through Google Tag Manager on a Zid store, using a GTM MCP server.
---

# ChatGPT Ads (OpenAI) Pixel via GTM — Zid Stores

## Overview

Every Zid store pushes the same GA4-shaped dataLayer, so the build is fixed: one base tag and four funnel events, each fed by exactly one Zid event. The work is to find the container, reuse what it already has, build in a separate workspace, verify, and stop. **Never publish.**

The Zid dataLayer, the OpenAI pixel rules and all tag code live in `reference.md` (the REFERENCE section below if this is a pasted prompt). Read it before Step 2.

| OpenAI event | Zid dataLayer event | Sends |
|---|---|---|
| `page_viewed` | every page load | — |
| `contents_viewed` | `view_item` | product, amount, currency |
| `items_added` | `add_to_cart` | product, amount, currency |
| `checkout_started` | `begin_checkout` | cart products, amount, currency |
| `order_created` | `purchase` | products, amount paid, currency, `event_id` = order ID |

Signed-in customers are matched by email and phone, hashed in the browser before anything is sent (reference section 5.1).

## Step 0 — Inputs gate

```
GTM account ............ <account ID or name>
GTM container .......... <GTM-XXXXXXX or container name>
OpenAI Pixel ID ........ <from Ads Manager → Conversions>
Store URL (optional) ... <https://...>
```

- Account/container missing → ask and **stop**.
- Pixel ID missing → ask once. If the user says to build without it, store the literal `REPLACE_WITH_OPENAI_PIXEL_ID` in the constant and say so first in the report.

Resolve the container by listing accounts, then the containers in that account. Names given by people are approximate ("acme store under agency new" → account `Agency - new`, container `Acme - store`). One clear match → state it and continue. Zero or several plausible matches → ask. If a store URL and a browser tool are available, confirm the container's public ID (`GTM-XXXX`) is the one loaded on the store.

## Step 1 — Load the reference

Read `reference.md`. If you have web access, also open `https://developers.openai.com/ads/measurement-pixel` and `https://developers.openai.com/ads/supported-events`. Where the live docs differ from the reference, the live docs win — say what changed.

## Step 2 — Inventory the container, reuse first

List workspaces, folders, triggers, variables and tags. Every list returns 20 items per page — read `totalPages` and fetch every page before deciding something doesn't exist. Template and quick-preview results can be very large: save them and search for `"name"`, `"type"`, `firingTriggerId`.

- **Triggers**: for `view_item`, `add_to_cart`, `begin_checkout` and `purchase`, look for a Custom Event trigger whose event filter equals that exact name. If one exists, reuse its ID whatever it is called. Create only the missing ones.
- **Variables**: for each dataLayer path in reference section 3, look for a Data Layer Variable on that same path. The Meta, Snap and TikTok Zid builds create the same `EA - Core - …` names; older builds use `ES - …` names. If one exists, use its real name inside the code in place of the `EA - Core - …` name. Create only the missing ones.
- An existing OpenAI tag (`oaiq` in a Custom HTML tag) means the pixel is already there: report it and ask before adding anything.
- Never edit, rename or delete anything that was already in the container.

**Live-site check (when a browser tool and the store URL are available).** Read the GTM script `src`: Zid stores often load `gtm.js?id=GTM-XXXX&l=gtmDataLayer`, so Zid's events are in `window.gtmDataLayer`, not `window.dataLayer`. Inspect the array named in `l=`. Check whether `window.oaiq` exists before GTM fires — a second install would double-count.

## Step 3 — Build (GTM MCP)

1. Create a dedicated workspace `EA - OpenAI Pixel`. Never build in the default workspace.
2. One atomic `bulkUpdate`, using temporary IDs (`new_1`, `new_2`, …) so entities can reference each other:
   - Folder `EA - Core` (if missing) — the shared Zid variables and triggers that were not already in the container.
   - Folder `EA - OpenAI` — everything OpenAI-specific.
   - Constant `EA - OpenAI - pixel_id` = the Pixel ID. **The ID appears nowhere else.**
   - The variables and five Custom HTML tags from reference sections 3–5.

| Entity | Name |
|---|---|
| Tags | `EA - OpenAI - init + page_viewed`, `EA - OpenAI - contents_viewed`, `EA - OpenAI - items_added`, `EA - OpenAI - checkout_started`, `EA - OpenAI - order_created` |
| OpenAI variables | `EA - OpenAI - pixel_id`, `EA - OpenAI - minor_unit_factor`, `EA - OpenAI - amount`, `EA - OpenAI - contents` |
| Shared Zid variables | `EA - Core - ecommerce_items`, `EA - Core - ecommerce_currency`, `EA - Core - ecommerce_value`, `EA - Core - ecommerce_transaction_id`, `EA - Core - customer_email`, `EA - Core - customer_mobile` |
| New triggers (only if none existed) | named after the dataLayer event: `view_item`, `add_to_cart`, `begin_checkout`, `purchase` |
| Folders / workspace | `EA - Core`, `EA - OpenAI` / `EA - OpenAI Pixel` |

If the MCP server has no `bulkUpdate`, create the same entities one at a time in this order — folders, variables, triggers, base tag, event tags — and set `parentFolderId` at creation.

Tag settings:
- `EA - OpenAI - init + page_viewed`: trigger All Pages (built-in ID `2147479553`), firing option **once per page**.
- The four event tags: firing option **once per event**, and the base tag as **setup tag** with `stopOnSetupFailure: true`. The setup tag defines `window.eaOaiq.identify`, which the event tags call.
- Put a `notes` line on every entity saying what it fires on and why.

## Step 4 — Verify, then stop

1. `quickPreview` the workspace and search the result for `compilerError`. The result is large; search it, do not read it whole.
2. Re-read the five tags and check each against the checklist in reference section 6.
3. **Do not create a version. Do not publish.** Publishing is the user's call, every time.

If the user wants to test by hand, add `debug: true` to **both** `init` calls in the base tag (the plain one and the one inside `identify`), tell them the logs appear in the browser console, and remove it again when they finish. Never hand over or publish a tag with `debug: true`.

## Step 5 — Report

Container and workspace · the five tags with the trigger each uses (reused or new) · variables reused vs created · whether the Pixel ID is real or a placeholder · the live-site check results · that a COD store (`payment_type: "zid_cod"`) counts orders at placement, not payment · what was not built and why (reference section 7) · the Preview checklist: product page → add to cart → checkout → one test order as a signed-in customer, checking that amounts are in minor units, `order_created` carries the order ID as `event_id`, and an `init` with `user` hashes appears before the event · "nothing is published".

## Common mistakes

| Mistake | Fix |
|---|---|
| Sending `amount: 74.25` for 74.25 SAR | Integer **minor units**: `7425`. Use the amount variables. |
| Recomputing the purchase amount from items | `ecommerce.value` is the amount paid (tax + shipping included). Items are the fallback only when `value` is missing. |
| Using the SKU as the product ID | `item_id` (fallback `id`). Zid names the SKU `item_sku` on most events and `sku` on `purchase`. |
| Reading `ecommerce.purchase.products` | Legacy UA block. Use `ecommerce.items`. |
| Passing `customer.email_hashed` / `mobile_hashed` to `user` | Their normalisation doesn't match OpenAI's on the sample checked. Hash the plain values through `eaOaiq.identify`. |
| Sending raw `customer.email` / `customer.mobile` anywhere | They go only into `eaOaiq.identify`, which hashes them in the browser. Hashes go in `init`, never in `measure`. |
| Hashing `+9665…` or `05…` | OpenAI wants the country code with no `+` and no leading zeros: `9665…`. A local number with no country code is skipped, not guessed. |
| `"{{EA - OpenAI - pixel_id}}"` in quotes inside Custom HTML | Write `{{EA - OpenAI - pixel_id}}` bare. |
| ES6 in GTM (`const`, `let`, arrow functions, template strings) | ES5 syntax only in Custom HTML and Custom JS variables. |
| `amount` without `currency` | Currency is required whenever amount is sent. The tags drop the amount if `ecommerce.currency` is missing. |
| Extra fields in `contents[]` (category, brand, sku, url) | Only `id`, `name`, `content_type`, `quantity`, `amount`, `currency`. |
| Event tags without the base tag as setup tag | `window.eaOaiq` is missing and the tag exits silently. |
| Inspecting `window.dataLayer` on a Zid store | Look at the array named in the GTM script's `l=` parameter, usually `gtmDataLayer`. |
| New trigger that duplicates an existing one | Reuse the existing trigger ID. |
| Missing items on page 2 of a list call | Lists return 20 per page. Fetch every page. |
| `debug: true` left in a tag | Remove it before handing over. |
| Publishing because "it compiled" | Stop at the workspace. Ask. |
