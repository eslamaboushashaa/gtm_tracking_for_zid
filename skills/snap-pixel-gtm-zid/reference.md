# Snap Pixel via GTM on Zid — Reference

Companion to `SKILL.md`. Read it before building. It holds the Zid dataLayer shape, the exact GTM MCP calls and payloads, and the post-publish checklist. "Me" / "I" in the steps below means the person who asked for the build.

Contents: 1. Zid dataLayer reference · 2. Build procedure (Steps 1–10) · 3. Post-publish checklist

---

## 1. Zid dataLayer reference

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

## 2. Build procedure — exact calls and payloads

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

## 3. Post-publish checklist (for the person who publishes)

- GTM Preview: every event fires once; `item_ids` resolves to a real array on both a 1-item and a 3-item cart.
- Snap Pixel Helper extension: `item_ids` shows as a list; `price` and `number_items` reflect the whole cart.
- Snapchat Ads Manager → Events Manager → Test Events: parameters populate.
- Edge case: an `ADD_CART` fired before the items push should send nothing rather than `"undefined"`.
- Confirm the Zid admin has no separate Snap Pixel field filled in.
