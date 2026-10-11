# TikTok Pixel via GTM on Zid — Reference

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
- `item_category` exists on **`view_item` only** → `content_category` is absent elsewhere. Expected.
- **No `item_brand` anywhere** → TikTok's `brand` field is never populated. Expected.
- SKU field is `item_sku` everywhere **except `purchase`, where it's `sku`**. Don't key on it.
- `ecommerce.value` (74.25) is far larger than Σ(price × quantity) (34) — it includes tax, shipping and fees. **Use `ecommerce.value`.** Keeps TikTok, Meta, Snap and Google Ads on one number.
- `payment_type: "zid_cod"` → **COD store.** See the PlaceAnOrder note below.
- Zid also fires **`view_cart`** (same shape as `begin_checkout`) and **`add_shipping_info`**, shown below. TikTok has no standard event for either — **do not build tags for them.** They are documented so you recognise them in Preview and don't read them as missing coverage.

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

**COD / `PlaceAnOrder`:** TikTok has a separate `PlaceAnOrder` standard event for "order placed, payment not settled". On a COD store, Zid's `purchase` fires at order placement — which is semantically `PlaceAnOrder`, not `CompletePayment`. Build `CompletePayment` as specified (it's the conventional GA4 `purchase` mapping and what the client's other platforms use), but **raise this in your report**: if COD volume is significant and a meaningful share of orders are never delivered, reported conversions will overstate real revenue. Splitting the two requires a Zid-side event that marks settlement, which does not exist today.


---

## 2. Build procedure — exact calls and payloads

### Step 1 — Locate the container
```
gtm_account list → gtm_container list → gtm_workspace list
```

### Step 2 — Get the TikTok template's real tag type
```
gtm_template action=list
```
Response is 100k+ chars and lands in a file. **Do not read it into context.** Parse it:
```python
import json
d = json.loads(open(PATH, encoding='utf-8').read())
for t in d['data']:
    gr = t.get('galleryReference', {})
    print(t['templateId'], '|', t['name'], '|', gr.get('owner'), '/', gr.get('repository'))
```
Find **TikTok Pixel** (owner `tiktok`, repo `gtm-template-pixel`), pull its `templateData`, read the `___INFO___` block. The `"id"` field is the tag `type` — currently **`cvt_MRQN8`**.

> ⚠️ The tag `type` is the template's OWN public ID, never `cvt_<containerId>_<templateId>`.

If it isn't installed, **stop and tell me** — I'll add it from the Community Template Gallery.

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

### Step 4 — Check for an existing TikTok install
Look for a Custom HTML tag containing `ttq.` or `analytics.tiktok.com`, another tag on the TikTok template, or a TikTok Pixel field filled in inside the Zid store admin.

### Step 5 — Folders
Create `EA - TikTok Pixel`. Create `EA - Core` if absent. Record both IDs.

### Step 6 — Variables

**Constant** (folder: TikTok)
```json
{"name": "EA - TikTok Pixel - pixel_id", "type": "c", "parentFolderId": "«TTFOLDER»",
 "notes": "TikTok Pixel ID. Single source of truth - the base tag and all 5 event tags reference this.",
 "parameter": [{"type": "template", "key": "value", "value": "<PIXEL ID>"}]}
```

**Advanced Matching DLVs** (folder: **Core**) — create only if missing:
| Name | dataLayer path |
|---|---|
| `EA - Core - customer_email_hashed` | `customer.email_hashed` |
| `EA - Core - customer_mobile_hashed` | `customer.mobile_hashed` |
| `EA - Core - customer_external_id` | `customer.external_id` |

**Two Custom JS variables** (folder: TikTok), `type: "jsm"`, key `javascript`. **ES5 only.**

`EA - TikTok Pixel - contents` — mirrors TikTok's own gtm-template-pixel conversion logic
```js
function() {
  var items = {{«ITEMS»}};
  if (!items || !items.length) {
    return undefined;
  }
  var contents = [];
  items.forEach(function(item) {
    if (!item) { return; }
    var id = item.item_id || item.id;
    if (!id) { return; }
    var content = {};
    content.content_id = String(id);
    content.content_type = 'product';
    if (item.item_name) { content.content_name = item.item_name; }
    if (item.item_brand) { content.brand = item.item_brand; }
    if (item.item_category) { content.content_category = item.item_category; }
    content.price = item.price ? Number(item.price) : 0;
    content.quantity = item.quantity ? Number(item.quantity) : 1;
    contents.push(content);
  });
  return contents.length ? contents : undefined;
}
```

`EA - TikTok Pixel - value`
```js
function() {
  var v = {{«VALUE»}};
  if (v !== undefined && v !== null && v !== '') {
    var n = Number(v);
    if (!isNaN(n)) { return Math.round(n * 100) / 100; }
  }
  var items = {{«ITEMS»}};
  if (!items || !items.length) { return undefined; }
  var total = 0;
  items.forEach(function(item) {
    var price = (item && item.price) ? Number(item.price) : 0;
    var qty = (item && item.quantity) ? Number(item.quantity) : 1;
    total += price * qty;
  });
  return Math.round(total * 100) / 100;
}
```

`contents` returning `undefined` rather than `[]` on an empty cart matters: an empty array reads to TikTok as "the cart is empty", which is a different claim from "no data yet".

### Step 7 — Create only the missing triggers
Custom Event triggers named after the dataLayer event:
```json
{"name": "add_payment_info", "type": "customEvent", "parentFolderId": "«TTFOLDER»",
 "customEventFilter": [{"type": "equals", "parameter": [
   {"type": "template", "key": "arg0", "value": "{{_event}}"},
   {"type": "template", "key": "arg1", "value": "add_payment_info"}]}]}
```
The base tag uses built-in trigger **`2147479553`** — never create one for it.

### Step 8 — Create the tags

**8a. Base tag** — `type: "html"`, `priority: 100`, trigger `2147479553`, folder TikTok.

```html
<script>
!function (w, d, t) {
  w.TiktokAnalyticsObject=t;var ttq=w[t]=w[t]||[];ttq.methods=["page","track","identify","instances","debug","on","off","once","ready","alias","group","enableCookie","disableCookie","holdConsent","revokeConsent","grantConsent"],ttq.setAndDefer=function(t,e){t[e]=function(){t.push([e].concat(Array.prototype.slice.call(arguments,0)))}};for(var i=0;i<ttq.methods.length;i++)ttq.setAndDefer(ttq,ttq.methods[i]);ttq.instance=function(t){for(var e=ttq._i[t]||[],n=0;n<ttq.methods.length;n++)ttq.setAndDefer(e,ttq.methods[n]);return e},ttq.load=function(e,n){var r="https://analytics.tiktok.com/i18n/pixel/events.js",o=n&&n.partner;ttq._i=ttq._i||{},ttq._i[e]=[],ttq._i[e]._u=r,ttq._t=ttq._t||{},ttq._t[e]=+new Date,ttq._o=ttq._o||{},ttq._o[e]=n||{},n=document.createElement("script"),n.type="text/javascript",n.async=!0,n.src=r+"?sdkid="+e+"&lib="+t,e=document.getElementsByTagName("script")[0],e.parentNode.insertBefore(n,e)};

  ttq.load('{{EA - TikTok Pixel - pixel_id}}');
  ttq.page();
}(window, document, 'ttq');
</script>
```

Set `supportDocumentWrite: false`. **Tell the client to diff this against TikTok Events Manager → Install Pixel → Install Code Manually** — this snippet is reproduced from the standard published base code, not pulled from their account.

**8b. Five event tags** — `type: "cvt_MRQN8"`, folder TikTok.

Base block on every one:
```json
{"type": "template", "key": "pixel_code",           "value": "{{EA - TikTok Pixel - pixel_id}}"},
{"type": "template", "key": "event",                "value": "<TikTokEvent>"},
{"type": "template", "key": "hash",                 "value": "hashed"},
{"type": "template", "key": "sha256_email",         "value": "{{EA - Core - customer_email_hashed}}"},
{"type": "template", "key": "sha256_phone",         "value": "{{EA - Core - customer_mobile_hashed}}"},
{"type": "template", "key": "sha256_external_id",   "value": "{{EA - Core - customer_external_id}}"},
{"type": "boolean",  "key": "enhance_ecomm",        "value": "false"},
{"type": "template", "key": "single_multi_product", "value": "multiple"},
{"type": "template", "key": "contents",             "value": "{{EA - TikTok Pixel - contents}}"},
{"type": "template", "key": "value",                "value": "{{EA - TikTok Pixel - value}}"},
{"type": "template", "key": "currency",             "value": "{{«CURRENCY»}}"}
```

Name each tag `EA - TikTok Pixel - <Tag>` (the base tag is `EA - TikTok Pixel - Base Code`).

| Tag | `event` | Trigger | Extra |
|---|---|---|---|
| ViewContent | `ViewContent` | `view_item` | — |
| AddToCart | `AddToCart` | `add_to_cart` | — |
| InitiateCheckout | `InitiateCheckout` | `begin_checkout` | — |
| AddPaymentInfo | `AddPaymentInfo` | `add_payment_info` | — |
| CompletePayment | `CompletePayment` | `purchase` | `order_id` = {{«TXNID»}}, `event_id` = {{«TXNID»}} |

TikTok has **no `payment_info_available` flag** (unlike Snap) — the AddPaymentInfo event firing is itself the signal.

`event_id` is TikTok's deduplication key against the server-side Events API. Setting it now means server-side dedup works later with no rework.

### Step 9 — Verify
```
gtm_folder action=entities folderId=«TTFOLDER»
```
Confirm 6 tags **and** the variables appear. **Variables created without `parentFolderId` will be missing** — fix with `gtm_folder action=moveEntitiesToFolder` passing the `variableId` array. Then confirm every `{{...}}` matches a real variable name.

### Step 10 — Report and stop
State: resolved IDs and template public ID; created vs reused; unverified assumptions; whether a second TikTok install exists; that the base snippet needs diffing against Events Manager; the COD/`PlaceAnOrder` caveat. **Do not publish.**


---

## 3. Post-publish checklist (for the person who publishes)

- GTM Preview: base tag fires **before** the event tags; each event fires once; `contents` resolves to a real array with one object per product on a multi-item cart.
- TikTok Pixel Helper: parameters land as expected.
- TikTok Events Manager → Test Events: `value`, `currency`, `order_id` populate.
- Edge case: `AddToCart` before the items push should send nothing, not `contents: []`.
- Confirm the Zid admin has no separate TikTok Pixel field filled in.
- Confirm `EA - TikTok Pixel - pixel_id` holds the real ID — a placeholder means `ttq.load('<garbage>')` on every pageview.
