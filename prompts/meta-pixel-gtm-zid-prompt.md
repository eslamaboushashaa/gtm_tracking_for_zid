# Meta Pixel via GTM — Zid Stores — Reusable Client Prompt

Paste everything below the line into Claude Code with the **GTM MCP server** connected.
Fill in the three INPUTS. Nothing else needs changing between clients.

---

You are setting up the Meta (Facebook) Pixel in a **Zid** store's Google Tag Manager web container, using the **GTM MCP server** for every read and write.

## INPUTS

```
GTM account ID or name .... <e.g. 1234567890  or  Client Name>
GTM container ID .......... <e.g. GTM-XXXXXXX>
Meta Pixel ID ............. <e.g. 123456789012345>
```

If any of the three is missing, ask for it and stop. Everything else in this document is fixed — do not ask me about platform, events, or field mapping.

---

## WHAT YOU ARE BUILDING (fixed spec)

**6 tags**, all using the installed facebook "Meta Pixel" gallery template:

| Tag name | Meta event | Fires on |
|---|---|---|
| `EA - Meta Pixel - PageView` | PageView | All Pages (built-in trigger `2147479553`) |
| `EA - Meta Pixel - ViewContent` | ViewContent | dataLayer `view_item` |
| `EA - Meta Pixel - AddToCart` | AddToCart | dataLayer `add_to_cart` |
| `EA - Meta Pixel - InitiateCheckout` | InitiateCheckout | dataLayer `begin_checkout` |
| `EA - Meta Pixel - AddPaymentInfo` | AddPaymentInfo | dataLayer `add_payment_info` |
| `EA - Meta Pixel - Purchase` | Purchase | dataLayer `purchase` |

Plus: **2 folders** (`EA - Meta Pixel` and the shared `EA - Core`), **1 constant**, **3 dataLayer variables**, **5 Custom JS variables**, and **only the triggers that don't already exist**.

Fixed decisions — do not re-litigate them:
- `content_ids` and `contents[].id` come from **`item_id`** (falling back to `id`). Flag in your final report that this must be verified against the client's Meta catalog, but build it this way. Never key on the SKU field — Zid names it `item_sku` on most events but `sku` on `purchase`.
- `value` always comes from **`ecommerce.value`**, never recomputed from items. See the purchase notes below.
- Advanced Matching sends **plain-text** `customer.email` and `customer.mobile`, never `email_hashed` / `mobile_hashed`. See Gotcha #1.
- `optInMetaCAPI` is set to `false` on every tag. See Gotcha #2.
- `consent` is set to `true` on every tag.
- Purchase carries `eventId` for future CAPI deduplication.

---

## ZID dataLayer REFERENCE

This is the real shape Zid pushes. Build against it.

```js
dataLayer.push({
  event: "view_item",
  customer: {
    is_guest: false,
    external_id: "<sha256>",
    email_hashed: "<sha256>",
    mobile_hashed: "<sha256>",
    email: "customer@example.com",
    mobile: "+201000000000"
  },
  ecommerce: {
    currency: "SAR",
    value: 11,
    items: [{
      id: "11111111-aaaa-4bbb-8ccc-222222222222",
      item_id: "11111111-aaaa-4bbb-8ccc-222222222222",
      item_sku: "1234567890123",
      price: 11,
      item_name: "منتج تجريبي",
      quantity: 1,
      item_category: "ألعاب وهدايا"
    }]
  }
});

dataLayer.push({
  event: "add_to_cart",
  customer: { /* same shape as above */ },
  ecommerce: {
    currency: "SAR",
    value: 11,
    items: [{
      id: "11111111-...", item_id: "11111111-...", item_sku: "1234567890123",
      price: 11, item_name: "...", quantity: 1
      // note: no item_category on add_to_cart
    }]
  }
});

dataLayer.push({
  event: "view_cart",
  customer: { /* same shape */ },
  ecommerce: { currency: "SAR", value: 11, items: [ /* same item shape */ ] }
});

dataLayer.push({
  event: "begin_checkout",
  customer: { /* same shape */ },
  ecommerce: { currency: "SAR", value: 11, items: [ /* same item shape */ ] }
});
```

**Key facts to build on:**
- `customer` sits at the **top level**, not inside `ecommerce`.
- Every item carries **both** `id` and `item_id` with the same UUID, plus `item_sku` (barcode).
- `item_category` appears on `view_item` only — the CJS variables must tolerate it being absent.
- `price` is per unit; `quantity` is separate. Meta's `contents[].item_price` = `price`.
- `ecommerce.value` is already the order/line total — pass it through, don't recompute.

```js
dataLayer.push({
  event: "add_payment_info",          // CONFIRMED: Zid does fire this
  customer: { /* same shape */ },
  ecommerce: {
    currency: "SAR",
    value: 74.25,
    payment_type: " - ",
    shipping_tier: "شيب قو - delivery",
    coupon: null,
    items: [
      { item_id: "11111111-...", item_name: "...", currency: "SAR",
        item_sku: "1234567890123", price: 11, quantity: 1 },
      { item_id: "33333333-...", item_name: "...", currency: "SAR",
        item_sku: "9876543210987", price: 23, quantity: 1 }
    ]
  }
});

dataLayer.push({
  event: "purchase",
  customer: { /* same shape */ },
  ecommerce: {
    transaction_id: 10012345,
    value: 74.25,                     // includes tax + shipping + fees
    tax: 9.68,
    shipping: 25,
    currency: "SAR",
    coupon: null,
    payment_type: "zid_cod",          // Cash on Delivery
    attribution: { utm_source: null, utm_medium: null, utm_campaign: null,
                   utm_term: null, utm_content: null, utm_id: null,
                   referer: null, visitor_id: null },
    items: [
      { affiliation: null, item_id: "11111111-...", item_name: "...",
        sku: "1234567890123",         // NOTE: "sku", not "item_sku"
        currency: "SAR", price: 11, quantity: 1,
        item_description: "المقاس :7×6×1.2سم" }
      // ...second item, no item_description
    ],
    purchase: { actionField: { /* legacy UA block */ }, products: [ /* ... */ ] }
  }
});
```

**Purchase-specific facts — these bite:**
- Items use **`sku`**, not `item_sku` like every other event. Irrelevant if you key on `item_id`, which you should.
- **No `item_category` at all** on purchase items. Any category variable resolves to undefined here.
- `item_description` appears on purchase items only.
- `ecommerce.value` (74.25) is **much larger than Σ(price × quantity)** (34). It includes tax, shipping and other fees. Send `ecommerce.value` — it matches what the customer paid and keeps Meta, Snap, TikTok and Google Ads reporting one number. Do not recompute from items.
- There is also a **legacy `ecommerce.purchase.products`** UA-style block alongside `items`. Ignore it; use `ecommerce.items`.
- `payment_type: "zid_cod"` — **this store is COD**. `purchase` fires at order placement, not payment settlement. Flag this: refunds/failed COD deliveries will inflate reported conversions.

**Two more events Zid fires that are NOT in scope.** Neither has a Meta standard event — do not build tags for them. Documented so you recognise them in Preview and don't mistake them for missing coverage:

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

`view_cart` carries the same shape as `begin_checkout`.

If the client later wants either tracked, the only options are a Meta **custom** event (`trackCustom`) or reusing an existing standard event — neither is a standard mapping, so raise it rather than deciding unilaterally.

---

## HARD RULES

1. **Never publish. Never create a container version.** Build in the workspace, verify, report, stop. Publishing is always my decision.
2. **Never create a trigger that already exists.** List triggers first; reuse any Custom Event trigger already matching the dataLayer event name, whatever it happens to be called (`View_item`, `add to cart`, etc.).
3. **Never rename or repurpose a pre-existing tag, trigger or variable.** Reuse them under their current names.
4. **Naming, no exceptions:**
   - Tags: `EA - Meta Pixel - <MetaEventName>` — Meta's event name, not the dataLayer name.
   - Meta-specific variables: `EA - Meta Pixel - <snake_case>`
   - Shared customer / ecommerce variables: `EA - Core - <snake_case>` (Snap and TikTok builds reuse them)
   - Folders: `EA - Meta Pixel` and `EA - Core`
5. **Pixel ID goes in exactly one Constant variable, `EA - Meta Pixel - pixel_id`.** Every tag references `{{EA - Meta Pixel - pixel_id}}`. Never hardcode the pixel ID in a tag.
6. **Everything you create goes in a folder** — tags, triggers and Meta-specific variables in `EA - Meta Pixel`; the shared `EA - Core - *` variables in `EA - Core`. Set `parentFolderId` at creation time.
7. Give every tag, trigger and variable a `notes` value stating what it does and any assumption baked into it.
8. Do all unblocked work first. Only ask me about something that would change the output and can't be discovered.

---

## PROCEDURE

### Step 1 — Locate the container

```
gtm_account   action=list
gtm_container action=list  accountId=<id>
gtm_workspace action=list  accountId=<id> containerId=<id>
```
Record `accountId`, `containerId`, `workspaceId`. Confirm `publicId` matches the GTM ID I gave you. If there are multiple workspaces, use the default one and say which you picked.

### Step 2 — Get the Meta Pixel template's real tag type

```
gtm_template action=list accountId=<id> containerId=<id> workspaceId=<id>
```

This response is ~120k chars and will be written to a file instead of returned. **Do not read that file into context.** Parse it:

```python
import json
d = json.loads(open(PATH, encoding='utf-8').read())
for t in d['data']:
    gr = t.get('galleryReference', {})
    print(t['templateId'], '|', t['name'], '|', gr.get('owner'), '/', gr.get('repository'))
```

Find the entry named **Meta Pixel** (gallery owner `facebook`, repo `GoogleTagManager-WebTemplate-For-FacebookPixel`). Then extract its `templateData` and read the `___INFO___` block:

```python
td = [t for t in d['data'] if t['templateId'] == '<id>'][0]['templateData']
i, j = td.find('___INFO___'), td.find('___TEMPLATE_PARAMETERS___')
print(td[i:j])
```

The `"id"` field in that block is the tag `type` you must use. For the current facebook template it is **`cvt_5RM3Q`**.

> ⚠️ **The single most common failure.** The tag `type` for a gallery template is the template's OWN public ID — NOT `cvt_<containerId>_<templateId>`. Guessing the latter returns:
> `Google API Error 400 - vendorTemplate.key: Unknown entity type (template public ID: cvt_...)`
> Verify it from `___INFO___` rather than assuming `cvt_5RM3Q`, in case the client has an older or forked copy.

If no Meta Pixel template is installed, **stop and tell me** — I'll install it from the Community Template Gallery.

### Step 3 — Audit what already exists

```
gtm_trigger  action=list accountId=<id> containerId=<id> workspaceId=<id>
gtm_variable action=list accountId=<id> containerId=<id> workspaceId=<id>
```

Produce a table before you write anything:

- **Triggers**: which of `view_item`, `add_to_cart`, `begin_checkout`, `add_payment_info`, `purchase` already have a Custom Event trigger, and their IDs. Reuse them.
- **Variables**: is there already a Data Layer Variable pointing at `ecommerce.items`, `ecommerce.currency`, `ecommerce.value`, `ecommerce.transaction_id`? Reuse them under their existing names.

Define these four placeholders for the rest of the build:

| Placeholder | If a DLV already exists | If not |
|---|---|---|
| `«ITEMS»` | its existing name | create `EA - Core - ecommerce_items` → `ecommerce.items` |
| `«CURRENCY»` | its existing name | create `EA - Core - ecommerce_currency` → `ecommerce.currency` |
| `«VALUE»` | its existing name | create `EA - Core - ecommerce_value` → `ecommerce.value` |
| `«TXNID»` | its existing name | create `EA - Core - ecommerce_transaction_id` → `ecommerce.transaction_id` |

State the resolved names in your report. Any of these you create go in `EA - Core`.

### Step 4 — Check for an existing pixel installation

Before building, look for a second install path that would double-fire every event:
- a Custom HTML tag whose `html` parameter contains `fbq(` or `connect.facebook.net`
- another tag already using the Meta Pixel template
- Zid's own Meta Pixel field in the store admin (I have to check that one — remind me)

If you find one, flag it prominently. Two installs means every event counts twice and Events Manager still looks green.

### Step 5 — Create the folders

```
gtm_folder action=create
createOrUpdateConfig = {
  "name": "EA - Meta Pixel",
  "notes": "All Meta (Facebook) Pixel tags, triggers and variables. Pixel ID lives in the constant {{EA - Meta Pixel - pixel_id}} - change it there, never in individual tags."
}
```
Record the returned `folderId` as `«FOLDER»`.

Then create `EA - Core` the same way if it doesn't already exist (notes: "Cross-platform variables shared by Meta, Snap and TikTok. Never rename anything in here - GTM resolves variables by name.") and record its ID as `«COREFOLDER»`.

### Step 6 — Create the variables

Meta-specific variables get `"parentFolderId": "«FOLDER»"`; the shared `EA - Core - *` variables get `"parentFolderId": "«COREFOLDER»"`.

**6a. Constant** — `gtm_variable action=create`
```json
{
  "name": "EA - Meta Pixel - pixel_id",
  "type": "c",
  "parentFolderId": "«FOLDER»",
  "notes": "Meta (Facebook) Pixel ID. Single source of truth - every Meta Pixel tag references this. To move the container to a different pixel, change this value only. Accepts a comma-separated list to fire multiple pixels.",
  "parameter": [
    {"type": "template", "key": "value", "value": "<PIXEL ID>"}
  ]
}
```

**6b. Three Data Layer Variables** — type `v`. Template for each:
```json
{
  "name": "EA - Core - customer_email",
  "type": "v",
  "parentFolderId": "«COREFOLDER»",
  "notes": "Plain-text email from dataLayer customer.email. Used for Meta Advanced Matching (em). The fbq library normalises + SHA-256 hashes this itself - do NOT pass the pre-hashed value here.",
  "parameter": [
    {"type": "integer",  "key": "dataLayerVersion", "value": "2"},
    {"type": "boolean",  "key": "setDefaultValue",  "value": "false"},
    {"type": "template", "key": "name",             "value": "customer.email"}
  ]
}
```
Repeat for:
| Name | dataLayer path | notes |
|---|---|---|
| `EA - Core - customer_email` | `customer.email` | as above |
| `EA - Core - customer_mobile` | `customer.mobile` | Plain-text phone. fbq normalises + hashes it itself. |
| `EA - Core - customer_external_id` | `customer.external_id` | Already SHA-256 hashed by Zid. Meta passes external_id through as-is — keep identical between Pixel and CAPI. |

**6c. Five Custom JavaScript variables** — type `jsm`, parameter key `javascript`, type `template`.

**ES5 ONLY.** No `let`/`const`, no arrow functions, no template literals, no `Array.map`/`reduce` with arrows. GTM will reject or silently misbehave otherwise.

Substitute `«ITEMS»` with the resolved items-variable name in all five.

`EA - Meta Pixel - content_ids` — *notes: "Maps ecommerce.items -> Meta content_ids (array of item_id UUIDs). These must match the retailer IDs in the client's Meta catalog."*
```js
function() {
  var items = {{«ITEMS»}};
  if (!items || !items.length) { return undefined; }
  var ids = [];
  for (var i = 0; i < items.length; i++) {
    var id = items[i].item_id || items[i].id;
    if (id) { ids.push(String(id)); }
  }
  return ids.length ? ids : undefined;
}
```

`EA - Meta Pixel - contents` — *notes: "Maps ecommerce.items -> Meta contents: [{id, quantity, item_price}]. Required for Advantage+ catalog ads."*
```js
function() {
  var items = {{«ITEMS»}};
  if (!items || !items.length) { return undefined; }
  var out = [];
  for (var i = 0; i < items.length; i++) {
    var it = items[i];
    var id = it.item_id || it.id;
    if (!id) { continue; }
    out.push({
      id: String(id),
      quantity: Number(it.quantity) || 1,
      item_price: Number(it.price) || 0
    });
  }
  return out.length ? out : undefined;
}
```

`EA - Meta Pixel - num_items` — *notes: "Sum of ecommerce.items[].quantity -> Meta num_items. Used on InitiateCheckout and Purchase."*
```js
function() {
  var items = {{«ITEMS»}};
  if (!items || !items.length) { return undefined; }
  var n = 0;
  for (var i = 0; i < items.length; i++) {
    n += Number(items[i].quantity) || 0;
  }
  return n;
}
```

`EA - Meta Pixel - content_name` — *notes: "First item's item_name -> Meta content_name. Used on ViewContent."*
```js
function() {
  var items = {{«ITEMS»}};
  if (!items || !items.length) { return undefined; }
  return items[0].item_name;
}
```

`EA - Meta Pixel - content_category` — *notes: "First item's item_category -> Meta content_category. Used on ViewContent. Zid only sends item_category on view_item."*
```js
function() {
  var items = {{«ITEMS»}};
  if (!items || !items.length) { return undefined; }
  return items[0].item_category;
}
```

Each returns `undefined` when there's nothing to send, so the template omits the property rather than sending an empty array.

### Step 7 — Create only the missing triggers

For each of the five dataLayer events with no existing trigger, `gtm_trigger action=create`:
```json
{
  "name": "add_payment_info",
  "type": "customEvent",
  "parentFolderId": "«FOLDER»",
  "notes": "Created for Meta Pixel AddPaymentInfo. Verify in GTM Preview that Zid actually pushes an add_payment_info event on this store - if it does not, this tag will never fire.",
  "customEventFilter": [{
    "type": "equals",
    "parameter": [
      {"type": "template", "key": "arg0", "value": "{{_event}}"},
      {"type": "template", "key": "arg1", "value": "add_payment_info"}
    ]
  }]
}
```
Name new triggers after the dataLayer event (`add_payment_info`, not `EA - add_payment_info`) so they sit consistently with any triggers already in the container.

PageView uses built-in trigger **`2147479553`** — never create one for it.

### Step 8 — Create the six tags

`gtm_tag action=create`, `"type": "<template public ID from Step 2>"`.

Every tag shares this **base block**. Only `name`, `standardEventName`, `firingTriggerId` and `objectPropertyList` change.

```json
{
  "name": "EA - Meta Pixel - <EventName>",
  "type": "cvt_5RM3Q",
  "parentFolderId": "«FOLDER»",
  "notes": "<see per-tag notes below>",
  "firingTriggerId": ["<trigger id>"],
  "parameter": [
    {"type": "template", "key": "pixelId",           "value": "{{EA - Meta Pixel - pixel_id}}"},
    {"type": "boolean",  "key": "enhancedEcommerce", "value": "false"},
    {"type": "boolean",  "key": "useGA4Ecommerce",   "value": "false"},
    {"type": "template", "key": "eventName",         "value": "standard"},
    {"type": "template", "key": "standardEventName", "value": "<MetaEventName>"},
    {"type": "template", "key": "consent",           "value": "true"},
    {"type": "boolean",  "key": "advancedMatching",  "value": "true"},
    {"type": "list", "key": "advancedMatchingList", "list": [
      {"type": "map", "map": [
        {"type": "template", "key": "name",  "value": "em"},
        {"type": "template", "key": "value", "value": "{{EA - Core - customer_email}}"}]},
      {"type": "map", "map": [
        {"type": "template", "key": "name",  "value": "ph"},
        {"type": "template", "key": "value", "value": "{{EA - Core - customer_mobile}}"}]},
      {"type": "map", "map": [
        {"type": "template", "key": "name",  "value": "external_id"},
        {"type": "template", "key": "value", "value": "{{EA - Core - customer_external_id}}"}]}
    ]},
    {"type": "template", "key": "objectPropertiesFromVariable", "value": "false"},
    {"type": "list", "key": "objectPropertyList", "list": [ /* per-tag, see below */ ]},
    {"type": "boolean", "key": "optInMetaCAPI",     "value": "false"},
    {"type": "boolean", "key": "dpoLDU",            "value": "false"},
    {"type": "boolean", "key": "disableAutoConfig", "value": "false"},
    {"type": "boolean", "key": "disablePushState",  "value": "false"}
  ]
}
```

Each `objectPropertyList` row has this shape:
```json
{"type": "map", "map": [
  {"type": "template", "key": "name",  "value": "content_ids"},
  {"type": "template", "key": "value", "value": "{{EA - Meta Pixel - content_ids}}"}]}
```

**Per-tag configuration:**

**1. `EA - Meta Pixel - PageView`**
- `standardEventName`: `PageView`
- `firingTriggerId`: `["2147479553"]`
- `objectPropertyList`: omit the key entirely (no object properties)
- notes: *"Base Meta Pixel + PageView on All Pages. Advanced Matching included; on a cold page load customer.* may be empty, which is fine - the template re-inits the pixel with matching data on the first later tag that carries it."*

**2. `EA - Meta Pixel - ViewContent`** — trigger: `view_item`
| name | value |
|---|---|
| `content_type` | `product` |
| `content_ids` | `{{EA - Meta Pixel - content_ids}}` |
| `contents` | `{{EA - Meta Pixel - contents}}` |
| `content_name` | `{{EA - Meta Pixel - content_name}}` |
| `content_category` | `{{EA - Meta Pixel - content_category}}` |
| `currency` | `{{«CURRENCY»}}` |
| `value` | `{{«VALUE»}}` |

notes: *"Fires on dataLayer event view_item. contents is required for Advantage+ catalog ads."*

**3. `EA - Meta Pixel - AddToCart`** — trigger: `add_to_cart`
`content_type`=`product`, `content_ids`, `contents`, `currency`, `value`
notes: *"Fires on dataLayer event add_to_cart. contents is required for Advantage+ catalog ads."*

**4. `EA - Meta Pixel - InitiateCheckout`** — trigger: `begin_checkout`
`content_type`=`product`, `content_ids`, `contents`, `num_items` (`{{EA - Meta Pixel - num_items}}`), `currency`, `value`
notes: *"Fires on dataLayer event begin_checkout. num_items is the summed quantity across ecommerce.items."*

**5. `EA - Meta Pixel - AddPaymentInfo`** — trigger: `add_payment_info`
`content_type`=`product`, `content_ids`, `contents`, `currency`, `value`
notes: *"Fires on dataLayer event add_payment_info. Confirmed present on Zid - carries currency, value, payment_type, shipping_tier and full items array."*

**6. `EA - Meta Pixel - Purchase`** — trigger: `purchase`
`content_type`=`product`, `content_ids`, `contents`, `num_items`, `currency`, `value`
**Plus one extra top-level parameter:**
```json
{"type": "template", "key": "eventId", "value": "{{«TXNID»}}"}
```
notes: *"Fires on dataLayer event purchase. currency and value are REQUIRED by Meta for this event. Event ID is set to ecommerce.transaction_id so browser events can be deduplicated against Conversions API events later."*

Meta requires `currency` and `value` on Purchase. `contents` is required for Advantage+ catalog ads on ViewContent / AddToCart / Purchase.

### Step 9 — Verify

```
gtm_folder action=entities folderId=«FOLDER»
gtm_folder action=entities folderId=«COREFOLDER»
```
Confirm across both folders: 6 tags, 9 variables (or fewer if you reused existing ecommerce DLVs), and the triggers you created. Then check that **every `{{...}}` string inside every tag matches a variable that actually exists** — GTM resolves variables by name and gives no error for a typo, the value just comes through undefined at runtime.

### Step 10 — Report and stop

State plainly:
1. Resolved account / container / workspace / folder IDs, and the template public ID you used.
2. What you **created** vs what you **reused** (name and ID for each).
3. Which assumptions are unverified: the `purchase` and `add_payment_info` dataLayer shapes, and that `item_id` matches the Meta catalog's retailer IDs.
4. Whether you found a pre-existing pixel installation.
5. What I need to check in GTM Preview before publishing.

**Then stop. Do not publish.**

---

## CRITICAL GOTCHAS

**1. Plain text vs hashed PII — silent failure.**
The Meta Pixel template passes Customer Information Data Parameters straight into `fbq('init', pixelId, params)` with **no hashing of its own** — verified by reading the template's sandboxed JS. The browser `fbq` library normalises and SHA-256 hashes plain `em` and `ph` itself, and plain values are the path Meta documents. Zid's dataLayer offers both `email` and `email_hashed`; **use the plain one**. Sending a pre-hashed value through `fbq('init')` is not a documented path: whether it still matches depends on Zid's normalisation being identical to Meta's, and a mismatch fails silently while Events Manager still shows the event as received. `external_id` is the exception — Meta treats it as an opaque ID, so Zid's already-hashed value is fine there as long as the same value is used everywhere.

**2. `optInMetaCAPI` defaults to `true`.**
Its label reads: *"You hereby authorize and instruct Meta to set up a Meta-enabled Conversions API integration on your behalf."* That's an authorisation the client hasn't given. Always set it explicitly to `false`.

**3. GTM resolves variables by NAME, not ID.**
Renaming a variable through the API does **not** rewrite the `{{...}}` strings inside tags. If you ever rename, you must update every referencing tag in the same pass. This is why the naming convention has to be applied at creation time, not as a cleanup pass.

**4. `update` replaces the entire entity.**
Any field you omit is deleted — including `parentFolderId`, `firingTriggerId`, and `notes`. Always `get` first (or reuse the full object from the `create` response) and send it back complete with your change applied. You also need the current `fingerprint`, which changes on every write.

**5. Two MCP responses blow the token limit.**
`gtm_template list` (~120k chars) and `gtm_workspace getStatus` (~150k chars) get written to a file instead of returned. Parse those with a script; never read them wholesale.

**6. `content_ids` correctness is invisible.**
If the IDs don't match the Meta catalog's retailer IDs, Events Manager stays green while Advantage+ catalog ads and dynamic remarketing quietly fail. Always flag this for manual verification in Commerce Manager → Catalog → Items.

**7. PageView fires before customer data exists.**
On a cold page load `customer.*` is empty, so the PageView tag sends no advanced matching. That's expected — the template re-initialises the pixel with matching data on the first later tag that carries it.

**8. `moveEntitiesToFolder` exists.**
`gtm_folder action=moveEntitiesToFolder` takes `tagId` / `triggerId` / `variableId` arrays. Cheaper than updating `parentFolderId` on each entity individually — but setting `parentFolderId` at creation time is cheaper still.

---

## POST-PUBLISH CHECKLIST (for me, not you)

- GTM Preview: each event fires exactly once; `contents` and `content_ids` resolve to real arrays, not `undefined`.
- Meta Pixel Helper: no warnings on a live product page.
- Events Manager → Test Events: correct `value` and `currency` arrive.
- Events Manager → event detail: Advanced Matching shows email / phone / external_id as **matched**, not merely sent.
- Commerce Manager → Catalog → Items: a `content_ids` value actually resolves to a product.
- Confirm Zid's own Meta Pixel field in the store admin is **empty**, or the pixel double-fires.
