# Snap Pixel via GTM — Zid Stores — Reusable Client Prompt

Paste everything below the line into Claude Code with the **GTM MCP server** connected.
Fill in the three INPUTS. Nothing else needs changing between clients.

---

You are setting up the Snapchat Pixel in a **Zid** store's Google Tag Manager web container, using the **GTM MCP server** for every read and write.

## INPUTS

```
GTM account ID or name .... <e.g. 1234567890  or  Client Name>
GTM container ID .......... <e.g. GTM-XXXXXXX>
Snap Pixel ID ............. <UUID, e.g. 12345678-abcd-1234-abcd-1234567890ab>
```

If any of the three is missing, ask for it and stop. Everything else here is fixed — do not ask me about platform, events, or field mapping.

---

## WHAT YOU ARE BUILDING (fixed spec)

**6 tags**, all on the installed Snapchat "Snap Pixel" gallery template:

| Tag name | Snap event_type | Fires on |
|---|---|---|
| `EA - Snap Pixel - PAGE_VIEW` | `PAGE_VIEW` | All Pages (built-in trigger `2147479553`) |
| `EA - Snap Pixel - VIEW_CONTENT` | `VIEW_CONTENT` | dataLayer `view_item` |
| `EA - Snap Pixel - ADD_CART` | `ADD_CART` | dataLayer `add_to_cart` |
| `EA - Snap Pixel - START_CHECKOUT` | `START_CHECKOUT` | dataLayer `begin_checkout` |
| `EA - Snap Pixel - ADD_BILLING` | `ADD_BILLING` | dataLayer `add_payment_info` |
| `EA - Snap Pixel - PURCHASE` | `PURCHASE` | dataLayer `purchase` |

Plus **1 folder**, **1 constant**, **6 Custom JS variables**, and **only the triggers that don't already exist**.

**Snap's template loads the pixel itself** (`injectScript(gateway_url)` then `snaptr('init', ...)`), so — unlike TikTok — there is **no separate base Custom HTML tag**. The PAGE_VIEW tag on All Pages does the init.

Fixed decisions — do not re-litigate:
- `item_ids` is a real **array**, never a comma-joined string.
- Advanced Matching uses Zid's **pre-hashed** values via `user_hashed_email` / `user_hashed_phone_number`.
- `price` always comes from `ecommerce.value`, never recomputed. See the dataLayer notes.
- Never touch `gateway_url`. See Gotcha #2.

---

## ZID dataLayer REFERENCE

```js
// view_item — the ONLY event carrying item_category
dataLayer.push({
  event: "view_item",
  customer: {
    is_guest: false,
    external_id: "<sha256>",      // already SHA-256 by Zid
    email_hashed: "<sha256>",     // already SHA-256
    mobile_hashed: "<sha256>",    // already SHA-256
    email: "customer@example.com",
    mobile: "+201000000000"
  },
  ecommerce: {
    currency: "SAR", value: 11,
    items: [{ id: "11111111-...", item_id: "11111111-...",
              item_sku: "1234567890123", price: 11,
              item_name: "...", quantity: 1,
              item_category: "ألعاب وهدايا" }]
  }
});

// add_to_cart / view_cart / begin_checkout — same shape, NO item_category

// add_payment_info — CONFIRMED present on Zid
dataLayer.push({
  event: "add_payment_info",
  customer: { /* same */ },
  ecommerce: {
    currency: "SAR", value: 74.25,
    payment_type: " - ", shipping_tier: "شيب قو - delivery", coupon: null,
    items: [{ item_id: "11111111-...", item_name: "...", currency: "SAR",
              item_sku: "1234567890123", price: 11, quantity: 1 }, /* ... */]
  }
});

// purchase
dataLayer.push({
  event: "purchase",
  customer: { /* same */ },
  ecommerce: {
    transaction_id: 10012345,
    value: 74.25,              // includes tax + shipping + fees
    tax: 9.68, shipping: 25, currency: "SAR", coupon: null,
    payment_type: "zid_cod",   // Cash on Delivery
    attribution: { utm_source: null, /* ... */ },
    items: [{ affiliation: null, item_id: "11111111-...", item_name: "...",
              sku: "1234567890123",          // NOTE: "sku", not "item_sku"
              currency: "SAR", price: 11, quantity: 1,
              item_description: "المقاس :7×6×1.2سم" }, /* ... */],
    purchase: { actionField: {...}, products: [...] }   // legacy UA block, ignore
  }
});
```

**Facts that drive the build:**
- `customer` is **top-level**, not inside `ecommerce`.
- Every item has both `id` and `item_id` (same UUID). Key on `item_id`.
- `item_category` exists on **`view_item` only**.
- `item_description` exists on **`purchase` only**.
- SKU field is `item_sku` everywhere **except `purchase`, where it's `sku`**. Don't key on it.
- `ecommerce.value` (74.25) is far larger than Σ(price × quantity) (34) — it includes tax, shipping and fees. **Use `ecommerce.value`.** It's what the customer paid and keeps Snap, Meta, TikTok and Google Ads on one number.
- `payment_type: "zid_cod"` → COD store. `purchase` fires at order placement, not settlement. Flag that failed COD deliveries will inflate conversions.
- Zid also fires **`view_cart`** (same shape as `begin_checkout`) and **`add_shipping_info`**, shown below. Snap has no standard event for either — **do not build tags for them.** They are documented so you recognise them in Preview and don't read them as missing coverage.

```js
dataLayer.push({
  event: "add_shipping_info",
  customer: { /* same shape */ },
  ecommerce: {
    currency: "SAR",
    value: 74.25,
    shipping_tier: "شيب قو - delivery",
    coupon: null,
    payment_type: " - ",
    items: [
      { item_id: "11111111-...", item_name: "...", item_sku: "1234567890123",
        currency: "SAR", price: 11, quantity: 1 },
      { item_id: "33333333-...", item_name: "...", item_sku: "9876543210987",
        currency: "SAR", price: 23, quantity: 1 }
    ]
  }
});
```

---

## HARD RULES

1. **Never publish. Never create a container version.** Build, verify, report, stop.
2. **Never create a trigger that already exists.** List triggers first and reuse.
3. **Never rename a pre-existing tag, trigger or variable.**
4. **Naming:** tags `EA - Snap Pixel - <SNAP_EVENT_TYPE>` (Snap's own constants, e.g. `VIEW_CONTENT`); variables `EA - Snap Pixel - <snake_case>` (shared ones `EA - Core - <snake_case>`); folder `EA - Snap Pixel`.
5. **Pixel ID lives only in the constant `EA - Snap Pixel - pixel_id`.** Every tag references `{{EA - Snap Pixel - pixel_id}}`.
6. **Everything you create goes in the `EA - Snap Pixel` folder** — set `parentFolderId` **at creation time** on tags AND variables. It is easy to forget on variables; check it in Step 9.
7. Shared customer variables belong in a separate **`EA - Core`** folder, not the Snap folder — Meta and TikTok use the same ones.
8. Give every entity a `notes` value.

---

## PROCEDURE

### Step 1 — Locate the container
```
gtm_account list → gtm_container list → gtm_workspace list
```
Record `accountId`, `containerId`, `workspaceId`. Confirm `publicId` matches.

### Step 2 — Get the Snap template's real tag type
```
gtm_template action=list
```
The response is 100k+ chars and lands in a file. **Do not read it into context.** Parse it:
```python
import json
d = json.loads(open(PATH, encoding='utf-8').read())
for t in d['data']:
    gr = t.get('galleryReference', {})
    print(t['templateId'], '|', t['name'], '|', gr.get('owner'))
```
Find **Snap Pixel** (gallery owner `Snapchat`, repo `snapchat-google-tag-manager`), pull its `templateData`, and read the `___INFO___` block. The `"id"` field is the tag `type`. For the current template it is **`cvt_K4VXG`**.

> ⚠️ The tag `type` is the template's OWN public ID — never `cvt_<containerId>_<templateId>`. Guessing returns
> `Google API Error 400 - vendorTemplate.key: Unknown entity type`.

If Snap Pixel isn't installed, **stop and tell me** — I'll add it from the Community Template Gallery (Templates → Search Gallery → "Snap Pixel" by Snap Inc.).

### Step 3 — Audit existing triggers and variables
```
gtm_trigger list ; gtm_variable list
```
Note which of `view_item` / `add_to_cart` / `begin_checkout` / `add_payment_info` / `purchase` already have a Custom Event trigger, and whether DLVs already exist for `ecommerce.items`, `ecommerce.currency`, `ecommerce.value`, `ecommerce.transaction_id`. Define:

| Placeholder | If a DLV exists | If not |
|---|---|---|
| `«ITEMS»` | its name | create `EA - Core - ecommerce_items` → `ecommerce.items` |
| `«CURRENCY»` | its name | create `EA - Core - ecommerce_currency` → `ecommerce.currency` |
| `«VALUE»` | its name | create `EA - Core - ecommerce_value` → `ecommerce.value` |
| `«TXNID»` | its name | create `EA - Core - ecommerce_transaction_id` → `ecommerce.transaction_id` |

### Step 4 — Check for an existing Snap install
Look for a Custom HTML tag containing `snaptr(`, another tag on the Snap template, or a Snap Pixel field filled in inside the Zid store admin. Two installs = every event counted twice.

### Step 5 — Folders
Create `EA - Snap Pixel`. Create `EA - Core` if it doesn't exist. Record both IDs.

### Step 6 — Variables

**Constant** (folder: Snap)
```json
{"name": "EA - Snap Pixel - pixel_id", "type": "c", "parentFolderId": "«SNAPFOLDER»",
 "notes": "Snapchat Pixel ID. Single source of truth - every Snap tag references this.",
 "parameter": [{"type": "template", "key": "value", "value": "<PIXEL ID>"}]}
```

**Advanced Matching DLVs** (folder: **Core**) — create only if missing:
| Name | dataLayer path |
|---|---|
| `EA - Core - customer_email_hashed` | `customer.email_hashed` |
| `EA - Core - customer_mobile_hashed` | `customer.mobile_hashed` |

DLV shape: `type: "v"`, params `dataLayerVersion: 2` (integer), `setDefaultValue: false` (boolean), `name` (template).

**Six Custom JS variables** (folder: Snap), `type: "jsm"`, parameter key `javascript`. **ES5 only** — `var`, `function()`, `.forEach` is fine; no arrow functions, `let`/`const`, or template literals. Substitute `«ITEMS»` and `«VALUE»`.

`EA - Snap Pixel - item_ids`
```js
function() {
  var products = {{«ITEMS»}};
  if (!products || !products.length) { return undefined; }
  var ids = [];
  products.forEach(function(prod) {
    var id = prod && (prod.item_id || prod.id);
    if (id) { ids.push(String(id)); }
  });
  return ids.length ? ids : undefined;
}
```

`EA - Snap Pixel - num_items` — total quantity, not distinct product count
```js
function() {
  var products = {{«ITEMS»}};
  if (!products || !products.length) { return undefined; }
  var total = 0;
  products.forEach(function(prod) {
    total += (prod && prod.quantity) ? Number(prod.quantity) : 1;
  });
  return total;
}
```

`EA - Snap Pixel - price` — prefers `ecommerce.value`, falls back to Σ(price × qty)
```js
function() {
  var v = {{«VALUE»}};
  if (v !== undefined && v !== null && v !== '') {
    var n = Number(v);
    if (!isNaN(n)) { return Math.round(n * 100) / 100; }
  }
  var products = {{«ITEMS»}};
  if (!products || !products.length) { return undefined; }
  var total = 0;
  products.forEach(function(prod) {
    var price = (prod && prod.price) ? Number(prod.price) : 0;
    var qty = (prod && prod.quantity) ? Number(prod.quantity) : 1;
    total += price * qty;
  });
  return Math.round(total * 100) / 100;
}
```

`EA - Snap Pixel - item_category` — Snap takes ONE category string, not a per-item array
```js
function() {
  var products = {{«ITEMS»}};
  if (!products || !products.length) { return undefined; }
  var counts = {};
  var best = null;
  var bestCount = 0;
  products.forEach(function(prod) {
    var cat = prod && prod.item_category;
    if (!cat) { return; }
    counts[cat] = (counts[cat] || 0) + 1;
    if (counts[cat] > bestCount) { bestCount = counts[cat]; best = cat; }
  });
  return best || undefined;
}
```

`EA - Snap Pixel - description` — first non-empty item description
```js
function() {
  var items = {{«ITEMS»}};
  if (!items || !items.length) { return undefined; }
  for (var i = 0; i < items.length; i++) {
    var it = items[i];
    var d = it && (it.item_description || it.description);
    if (d) { return String(d); }
  }
  return undefined;
}
```

`EA - Snap Pixel - payment_info_available` — numeric 1
```js
function() {
  return 1;
}
```

Every one returns `undefined` rather than an empty array/string when there's nothing to send, so Snap omits the property instead of receiving `"undefined"` as a literal.

### Step 7 — Create only the missing triggers
Custom Event triggers named after the dataLayer event:
```json
{"name": "add_payment_info", "type": "customEvent", "parentFolderId": "«SNAPFOLDER»",
 "customEventFilter": [{"type": "equals", "parameter": [
   {"type": "template", "key": "arg0", "value": "{{_event}}"},
   {"type": "template", "key": "arg1", "value": "add_payment_info"}]}]}
```
PAGE_VIEW uses built-in trigger **`2147479553`**.

### Step 8 — Create the six tags

`gtm_tag action=create`, `"type": "cvt_K4VXG"` (or whatever Step 2 returned), `"parentFolderId": "«SNAPFOLDER»"`.

**Base block on every tag:**
```json
{"type": "template", "key": "pixel_id",                 "value": "{{EA - Snap Pixel - pixel_id}}"},
{"type": "template", "key": "event_type",               "value": "<SNAP_EVENT>"},
{"type": "template", "key": "user_hashed_email",        "value": "{{EA - Core - customer_email_hashed}}"},
{"type": "template", "key": "user_hashed_phone_number", "value": "{{EA - Core - customer_mobile_hashed}}"},
{"type": "boolean",  "key": "enable_console_logging",   "value": "false"}
```

**Per tag, add:**

Name each tag `EA - Snap Pixel - <Tag>`.

| Tag | event_type | Trigger | Extra parameters |
|---|---|---|---|
| PAGE_VIEW | `PAGE_VIEW` | `2147479553` | *(none)* |
| VIEW_CONTENT | `VIEW_CONTENT` | `view_item` | item_ids, item_category, description, price, number_items, currency |
| ADD_CART | `ADD_CART` | `add_to_cart` | item_ids, item_category, description, price, number_items, currency |
| START_CHECKOUT | `START_CHECKOUT` | `begin_checkout` | item_ids, item_category, description, price, number_items, currency |
| ADD_BILLING | `ADD_BILLING` | `add_payment_info` | item_ids, item_category, description, price, number_items, currency, **payment_info_available** |
| PURCHASE | `PURCHASE` | `purchase` | item_ids, item_category, description, price, number_items, currency, **transaction_id**, **client_dedup_id** |

Value mapping:
`item_ids`={{EA - Snap Pixel - item_ids}} · `item_category`={{EA - Snap Pixel - item_category}} · `description`={{EA - Snap Pixel - description}} · `price`={{EA - Snap Pixel - price}} · `number_items`={{EA - Snap Pixel - num_items}} · `currency`={{«CURRENCY»}} · `payment_info_available`={{EA - Snap Pixel - payment_info_available}} · `transaction_id` and `client_dedup_id` = {{«TXNID»}}

`item_category` and `description` are wired onto every event even though Zid only populates them on `view_item` and `purchase` respectively. They self-omit when empty, and populate automatically if Zid ever adds them.

`client_dedup_id` is Snap's deduplication key against the Conversions API. Setting it to the transaction ID now means server-side dedup works later with no rework.

### Step 9 — Verify
```
gtm_folder action=entities folderId=«SNAPFOLDER»
```
Confirm 6 tags **and** the variables are listed. **A variable created without `parentFolderId` will not appear** — if any are missing, fix with `gtm_folder action=moveEntitiesToFolder` passing the `variableId` array. Then confirm every `{{...}}` in every tag matches a real variable name.

### Step 10 — Report and stop
State: resolved IDs and template public ID; what you created vs reused; unverified assumptions; whether a second Snap install exists; what to check in Preview. **Do not publish.**

---

## CRITICAL GOTCHAS

**1. Snap has no `external_id` field.** Its Advanced Matching offers only email, phone, first/last name and mobile ad ID. Zid's `customer.external_id` is unusable here — it works on Meta and TikTok only. Don't invent a field for it.

**2. `gateway_url` — leave it completely alone.** It is labelled "CAPI Gateway Script URL" and has `defaultValue: "https://sc-static.net/scevent.min.js"` baked into the template. Do **not** set it. Two reasons: setting it to the default pins the URL so a future template update can't change it, and the field implies a Conversions API Gateway is configured when it isn't. Note for the client: GTM's UI repopulates the default the moment the field is cleared, so it will *look* set even when no value is stored. That's cosmetic — the stored parameter is what matters.

**3. Hashed vs plain PII differs per platform — don't "harmonise" them.** Snap and TikTok take Zid's pre-hashed values. **Meta gets the plain ones** — `fbq` hashes plain values itself, and plain values are the path Meta documents.

**4. `payment_info_available` accepts variables** (`macrosInSelect: true`), so `{{EA - Snap Pixel - payment_info_available}}` is valid — no need to hardcode `1`.

**5. `update` replaces the entire entity.** Omitted fields are deleted, including `parentFolderId`, `firingTriggerId` and `notes`. Always send the complete object plus the current `fingerprint`.

**6. GTM resolves variables by NAME.** Renaming one does not rewrite `{{...}}` references inside tags. Apply naming at creation time.

**7. The GTM API rate-limits hard.** Batches of ~5+ writes per minute trigger `429 Quota exceeded for quota metric 'Queries'`. Pace the writes; a 429 is transient, not a failure of the payload.

**8. `gtm_template list` and `gtm_workspace getStatus` overflow the token limit** and are written to a file. Parse with a script, never read wholesale.

---

## POST-PUBLISH CHECKLIST (mine, not yours)

- GTM Preview: every event fires once; `item_ids` resolves to a real array on both a 1-item and a 3-item cart.
- Snap Pixel Helper extension: `item_ids` shows as a list; `price` and `number_items` reflect the whole cart.
- Snapchat Ads Manager → Events Manager → Test Events: parameters populate.
- Edge case: an `ADD_CART` fired before the items push should send nothing rather than `"undefined"`.
- Confirm the Zid admin has no separate Snap Pixel field filled in.
