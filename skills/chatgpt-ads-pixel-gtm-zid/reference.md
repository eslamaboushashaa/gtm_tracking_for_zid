# REFERENCE — ChatGPT Ads (OpenAI) pixel on Zid

Pixel facts from `developers.openai.com/ads/measurement-pixel` and `/ads/supported-events`, read 2026-10-17. Zid dataLayer shapes captured from a live Zid store in 2026 (IDs and customer data below are fictional). The tag pattern — loader, `init`, `measure`, minor-unit amounts, setup tag — is the one built and tested in GTM Preview on a live Salla store on 2026-10-11; the Zid variables and the in-browser hashing are new and have not been run on a live Zid store yet.

There is no official GTM template for this pixel — every tag is Custom HTML.

## 1. OpenAI pixel rules

- `oaiq("init", { pixelId, user })` — `pixelId` required. `debug: true` logs SDK activity to the browser console; testing only.
- `oaiq("measure", eventName, eventData, options)` — `eventData.type` must match the event. `options.event_id` is the deduplication ID.
- Dedup key = pixel ID + event name + `event_id`. The first event wins, so an `event_id` also protects against a thank-you page reload.
- `oaiq("consent", false)` before `init` blocks measurement; `oaiq("consent", true)` after opt-in. Default is granted. Blocked events are not replayed.
- Automatic: stores the `oppref` click parameter in the `__oppref` cookie (30 days), sets `__obref` (365 days), adds `source_url` and a timestamp.
- CSP, if the store sets one: `script-src https://bzrcdn.openai.com`; `connect-src https://bzr.openai.com https://bzrcdn.openai.com`; `img-src https://bzr.openai.com`.

Events used on Zid — all have `type: "contents"`, with optional `amount`, `currency`, `contents[]`:

| Event | Meaning |
|---|---|
| `page_viewed` | page load |
| `contents_viewed` | product viewed |
| `items_added` | added to cart |
| `checkout_started` | checkout begins |
| `order_created` | purchase completed |

`contents[]` item — **only these fields**: `id` (string), `name`, `content_type` (`"product"`), `quantity` (integer), `amount` (price of one unit, integer minor units), `currency`.

**Amount rules**
- Integer in the currency's ISO 4217 minor unit: 74.25 SAR → `7425`; 5 KWD → `5000` (3 decimals); 500 JPY → `500` (0 decimals).
- Event `amount` = value of the whole event. Item `amount` = price of **one** unit; `quantity` is sent separately.
- `amount` present → `currency` required.

**User data** — in `init` only, never in `measure`. SHA-256, 64-character lowercase hex, never raw:

| Field | Normalise before hashing |
|---|---|
| `email_sha256` | trim, lowercase |
| `phone_number_sha256` | keep the country code; remove spaces, parentheses, periods, hyphens, the leading `+` and leading zeros; 8–15 digits (`+966 50 000 0000` → `966500000000`) |

Hash = SHA-256 of the UTF-8 normalised value. `user` also accepts `external_id_sha256`, `first_name_sha256`, `last_name_sha256` and plain `country`, `city`, `region`, `postal_code`; this build sends only email and phone (section 7).

To add user data later in the page, call `init` again with the complete `user` object — documented for exactly this case (data that appears after the first `init`).

## 2. Zid dataLayer

Zid pushes GA4-shaped events. `ecommerce` and `customer` are both **top-level** keys. Many Zid stores load GTM with a custom dataLayer name (`gtm.js?id=GTM-XXXX&l=gtmDataLayer`): the events then live in `window.gtmDataLayer`, not `window.dataLayer`. GTM listens to the right array either way, so triggers and variables work unchanged — only manual inspection in the console has to use the name in `l=`.

```js
// Product page
{ event: "view_item",
  customer: { is_guest: false, email: "customer@example.com", mobile: "+966500000000",
              email_hashed: "<64 hex>", mobile_hashed: "<64 hex>", external_id: "<64 hex>" },
  ecommerce: { currency: "SAR", value: 11,
    items: [{ id: "11111111-aaaa-4bbb-8ccc-222222222222", item_id: "11111111-aaaa-4bbb-8ccc-222222222222",
              item_sku: "1234567890123", item_name: "منتج تجريبي", price: 11, quantity: 1,
              item_category: "Sample category" }] } }

// Add to cart — same shape, no item_category
{ event: "add_to_cart", customer: { /* same */ },
  ecommerce: { currency: "SAR", value: 11, items: [ /* same item shape */ ] } }

// Checkout
{ event: "begin_checkout", customer: { /* same */ },
  ecommerce: { currency: "SAR", value: 34, items: [ /* one entry per cart line */ ] } }

// Thank-you page — the purchase source
{ event: "purchase", customer: { /* same */ },
  ecommerce: { transaction_id: 10012345, value: 74.25, tax: 9.68, shipping: 25, currency: "SAR",
    coupon: null, payment_type: "zid_cod",
    items: [
      { item_id: "11111111-aaaa-4bbb-8ccc-222222222222", item_name: "منتج تجريبي", sku: "1234567890123",
        currency: "SAR", price: 11, quantity: 1, item_description: "…" },
      { item_id: "33333333-aaaa-4bbb-8ccc-444444444444", item_name: "Second product", sku: "9876543210987",
        currency: "SAR", price: 23, quantity: 1 } ],
    purchase: { actionField: { /* legacy UA block */ }, products: [ /* … */ ] } } }

// Guest visitor — customer carries nothing else
{ customer: { is_guest: true } }
```

Facts that decide the build:

1. **Product ID = `item_id`** (fallback `id`) — the UUID. Never the SKU: Zid calls it `item_sku` on most events and `sku` on `purchase`.
2. **Event value = `ecommerce.value`**, passed through. On `purchase` it is the amount paid — tax, shipping and fees included — and is far larger than Σ(price × quantity). Never recompute it from items; items are the fallback only when `value` is missing.
3. **`price` is per unit**; `quantity` is separate.
4. **`ecommerce.items` is the only item list.** Ignore the legacy `ecommerce.purchase.products` block on `purchase`.
5. **`ecommerce.currency` is on every ecommerce push.** If it is ever missing, send no amount (OpenAI requires currency with every amount).
6. **Customer data comes with every event**, but only for a signed-in or identified customer. Guests carry `{ is_guest: true }` and nothing else, so `user` stays empty for them — expected.
7. **Zid's `email_hashed` / `mobile_hashed` are not used.** On the only sample checked, neither matched SHA-256 of the plain value under any normalisation tried (raw, lowercased, digits only, with or without `+`, local format). Zid's hashing method is unknown, and a wrong hash fails silently. The tags hash `customer.email` and `customer.mobile` in the browser with OpenAI's own normalisation instead (section 5.1). The raw values never leave the browser.
8. **`customer.external_id`** is a Zid-side hash of an unknown value. OpenAI's `external_id_sha256` only helps when the store's own server-side events send the same ID, and none exist — not sent.
9. **`payment_type: "zid_cod"`** means cash on delivery: `purchase` fires when the order is placed, not when it is paid. Undelivered orders still count as conversions. Say so in the report.

## 3. Shared Zid variables and triggers (folder `EA - Core`)

These names are shared with the Meta, Snap and TikTok Zid builds. Create only what the container does not already have (workflow Step 2). If a Data Layer Variable on the same path exists under another name, use that name in sections 4–5.

Data Layer Variables, Version 2, no default value:

| Name | Path |
|---|---|
| `EA - Core - ecommerce_items` | `ecommerce.items` |
| `EA - Core - ecommerce_currency` | `ecommerce.currency` |
| `EA - Core - ecommerce_value` | `ecommerce.value` |
| `EA - Core - ecommerce_transaction_id` | `ecommerce.transaction_id` |
| `EA - Core - customer_email` | `customer.email` |
| `EA - Core - customer_mobile` | `customer.mobile` |

Triggers — type Custom Event, event name **equals**, case-sensitive. Name new ones after the dataLayer event, the same as the other Zid builds:

| Name (if it has to be created) | Event name |
|---|---|
| `view_item` | `view_item` |
| `add_to_cart` | `add_to_cart` |
| `begin_checkout` | `begin_checkout` |
| `purchase` | `purchase` |

The base tag uses the built-in All Pages trigger `2147479553` — never create one.

## 4. OpenAI variables (folder `EA - OpenAI`) — ES5 only

Constant `EA - OpenAI - pixel_id` = the Pixel ID.

`EA - OpenAI - minor_unit_factor`:

```js
function() {
  var c = String({{EA - Core - ecommerce_currency}} || '').toUpperCase();
  var three = {BHD: 1, IQD: 1, JOD: 1, KWD: 1, LYD: 1, OMR: 1, TND: 1};
  var zero = {BIF: 1, CLP: 1, DJF: 1, GNF: 1, JPY: 1, KMF: 1, KRW: 1, PYG: 1, RWF: 1, UGX: 1, VND: 1, VUV: 1, XAF: 1, XOF: 1, XPF: 1};
  if (three[c]) { return 1000; }
  if (zero[c]) { return 1; }
  return 100;
}
```

`EA - OpenAI - amount` (`ecommerce.value` in minor units; items only as a fallback):

```js
function() {
  var v = Number({{EA - Core - ecommerce_value}});
  if (isNaN(v) || v <= 0) {
    v = 0;
    var items = {{EA - Core - ecommerce_items}};
    if (items && items.length) {
      for (var i = 0; i < items.length; i++) {
        var p = Number(items[i] && items[i].price);
        var q = parseInt(items[i] && items[i].quantity, 10) || 1;
        if (!isNaN(p) && p > 0) { v += p * q; }
      }
    }
  }
  if (!v) { return undefined; }
  return Math.round(v * {{EA - OpenAI - minor_unit_factor}});
}
```

`EA - OpenAI - contents`:

```js
function() {
  var items = {{EA - Core - ecommerce_items}};
  if (!items || !items.length) { return undefined; }
  var factor = {{EA - OpenAI - minor_unit_factor}};
  var eventCurrency = {{EA - Core - ecommerce_currency}};
  var out = [];
  for (var i = 0; i < items.length; i++) {
    var it = items[i];
    var id = it && (it.item_id || it.id);
    if (!id) { continue; }
    var item = {
      id: String(id),
      content_type: 'product',
      quantity: parseInt(it.quantity, 10) || 1
    };
    if (it.item_name) { item.name = String(it.item_name); }
    var price = Number(it.price);
    var currency = it.currency || eventCurrency;
    if (!isNaN(price) && price > 0 && currency) {
      item.amount = Math.round(price * factor);
      item.currency = String(currency);
    }
    out.push(item);
  }
  return out.length ? out : undefined;
}
```

## 5. Tags (folder `EA - OpenAI`) — Custom HTML, `supportDocumentWrite` false

GTM variables are written bare inside `<script>` — no quotes around `{{…}}`. ES5 syntax only: `Promise`, `crypto.subtle` and `TextEncoder` are browser APIs called with ordinary ES5 code, which GTM accepts.

### 5.1 `EA - OpenAI - init + page_viewed`

Trigger: All Pages (`2147479553`). Firing option: once per page.

Loads the SDK, calls `init`, sends `page_viewed`, and defines `window.eaOaiq.identify(email, phone, done)`: it normalises and SHA-256 hashes the plain values in the browser, re-calls `init` with the `user` object when the hashes are new, then runs `done`. `done` always runs — immediately when there is nothing to hash or the browser lacks Web Crypto, and after at most one second if hashing stalls — so an event is never lost.

```html
<script>
  (function (w, d, s, u) {
    if (w.oaiq) return;
    var q = function () { q.q.push(arguments); };
    q.q = [];
    w.oaiq = q;
    var js = d.createElement(s);
    js.async = true;
    js.src = u;
    var f = d.getElementsByTagName(s)[0];
    f.parentNode.insertBefore(js, f);
  })(window, document, "script", "https://bzrcdn.openai.com/sdk/oaiq.min.js");

  (function (w) {
    var pixelId = {{EA - OpenAI - pixel_id}};
    if (!w.eaOaiq) {
      var ea = { pixelId: pixelId, userKey: "" };
      var hex = function (buf) {
        var b = new Uint8Array(buf), out = "";
        for (var i = 0; i < b.length; i++) { out += (b[i] < 16 ? "0" : "") + b[i].toString(16); }
        return out;
      };
      var sha256 = function (str) {
        return w.crypto.subtle.digest("SHA-256", new w.TextEncoder().encode(str)).then(hex);
      };
      var normEmail = function (v) {
        if (typeof v !== "string") return "";
        v = v.replace(/^\s+|\s+$/g, "").toLowerCase();
        return /^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(v) ? v : "";
      };
      var normPhone = function (v) {
        if (v === undefined || v === null) return "";
        v = String(v).replace(/^\s+|\s+$/g, "");
        var digits = v.replace(/\D/g, "");
        if (v.charAt(0) === "+") {
          // international format: keep the digits
        } else if (v.indexOf("00") === 0) {
          digits = digits.slice(2);
        } else if (!/^[1-9]\d{10,14}$/.test(digits)) {
          return ""; // local format without a country code: do not guess
        }
        digits = digits.replace(/^0+/, "");
        return digits.length >= 8 && digits.length <= 15 ? digits : "";
      };
      ea.identify = function (email, phone, done) {
        var e = normEmail(email), p = normPhone(phone);
        var finished = false;
        var finish = function () { if (!finished) { finished = true; done(); } };
        if ((!e && !p) || !(w.crypto && w.crypto.subtle && w.TextEncoder && w.Promise)) { finish(); return; }
        setTimeout(finish, 1000);
        w.Promise.all([e ? sha256(e) : "", p ? sha256(p) : ""]).then(function (h) {
          var key = h[0] + "|" + h[1];
          if (key !== ea.userKey) {
            var user = {};
            if (h[0]) { user.email_sha256 = h[0]; }
            if (h[1]) { user.phone_number_sha256 = h[1]; }
            ea.userKey = key;
            w.oaiq("init", { pixelId: ea.pixelId, user: user });
          }
          finish();
        }, finish);
      };
      w.eaOaiq = ea;
    }
    w.oaiq("init", { pixelId: pixelId });
    w.oaiq("measure", "page_viewed", { type: "contents" });
  })(window);
</script>
```

### 5.2 `EA - OpenAI - contents_viewed`, `items_added`, `checkout_started`

Triggers: `view_item`, `add_to_cart`, `begin_checkout`. Same body; only the event name in the `measure` call changes (`"contents_viewed"` / `"items_added"` / `"checkout_started"`). Once per event, setup tag = 5.1.

```html
<script>
  (function () {
    var ea = window.eaOaiq;
    if (!window.oaiq || !ea) return;
    var data = { type: "contents" };
    var amount = {{EA - OpenAI - amount}};
    var currency = {{EA - Core - ecommerce_currency}};
    var contents = {{EA - OpenAI - contents}};
    if (amount && currency) {
      data.amount = amount;
      data.currency = String(currency);
    }
    if (contents) { data.contents = contents; }
    ea.identify({{EA - Core - customer_email}}, {{EA - Core - customer_mobile}}, function () {
      window.oaiq("measure", "contents_viewed", data);
    });
  })();
</script>
```

### 5.3 `EA - OpenAI - order_created`

Trigger: `purchase` only. Once per event, setup tag = 5.1. Same body plus the order ID as `event_id`.

```html
<script>
  (function () {
    var ea = window.eaOaiq;
    if (!window.oaiq || !ea) return;
    var data = { type: "contents" };
    var amount = {{EA - OpenAI - amount}};
    var currency = {{EA - Core - ecommerce_currency}};
    var contents = {{EA - OpenAI - contents}};
    if (amount && currency) {
      data.amount = amount;
      data.currency = String(currency);
    }
    if (contents) { data.contents = contents; }
    var orderId = {{EA - Core - ecommerce_transaction_id}};
    ea.identify({{EA - Core - customer_email}}, {{EA - Core - customer_mobile}}, function () {
      if (orderId) {
        window.oaiq("measure", "order_created", data, { event_id: String(orderId) });
      } else {
        window.oaiq("measure", "order_created", data);
      }
    });
  })();
</script>
```

### 5.4 GTM API shapes (for a GTM MCP server that takes raw entities)

```json
{ "variable": { "variableId": "new_2", "name": "EA - Core - ecommerce_value", "type": "v", "parentFolderId": "new_1",
  "parameter": [ { "type": "integer", "key": "dataLayerVersion", "value": "2" },
                 { "type": "boolean", "key": "setDefaultValue", "value": "false" },
                 { "type": "template", "key": "name", "value": "ecommerce.value" } ] } }

{ "variable": { "name": "EA - OpenAI - pixel_id", "type": "c",
  "parameter": [ { "type": "template", "key": "value", "value": "<PIXEL ID>" } ] } }

{ "variable": { "name": "EA - OpenAI - amount", "type": "jsm",
  "parameter": [ { "type": "template", "key": "javascript", "value": "function() { … }" } ] } }

{ "trigger": { "name": "purchase", "type": "customEvent",
  "customEventFilter": [ { "type": "equals", "parameter": [
      { "type": "template", "key": "arg0", "value": "{{_event}}" },
      { "type": "template", "key": "arg1", "value": "purchase" } ] } ] } }

{ "tag": { "name": "EA - OpenAI - order_created", "type": "html", "tagFiringOption": "oncePerEvent",
  "firingTriggerId": [ "<trigger id>" ],
  "setupTag": [ { "tagName": "EA - OpenAI - init + page_viewed", "stopOnSetupFailure": true } ],
  "parameter": [ { "type": "template", "key": "html", "value": "<script>…</script>" },
                 { "type": "boolean", "key": "supportDocumentWrite", "value": "false" } ] } }
```

Base tag: `"tagFiringOption": "oncePerLoad"`, `"firingTriggerId": ["2147479553"]`.

## 6. Verification checklist

- Workspace compiles: no `compilerError` in the quick preview.
- The Pixel ID appears only in `EA - OpenAI - pixel_id`.
- Five tags, each with exactly one trigger; `order_created` only on `purchase`.
- The four event tags have the base tag as setup tag.
- Every `amount` comes from `EA - OpenAI - amount` or the per-unit conversion inside `EA - OpenAI - contents` — no raw Zid value is sent as an amount.
- Every `measure` that sends `amount` also sends `currency`.
- `customer.email` and `customer.mobile` are read only by the event tags and only passed to `eaOaiq.identify`. No tag sends them, and nothing reads `customer.email_hashed`, `customer.mobile_hashed` or `customer.external_id`.
- No `debug: true`.
- Worked example for the samples in section 2:
  - `contents_viewed` → `{ type: "contents", amount: 1100, currency: "SAR", contents: [{ id: "11111111-aaaa-4bbb-8ccc-222222222222", content_type: "product", quantity: 1, name: "منتج تجريبي", amount: 1100, currency: "SAR" }] }`
  - `checkout_started` → `{ type: "contents", amount: 3400, currency: "SAR", contents: [ … ] }`
  - `order_created` → `{ type: "contents", amount: 7425, currency: "SAR", contents: [{ id: "11111111-…", …, amount: 1100 }, { id: "33333333-…", …, amount: 2300 }] }` with options `{ event_id: "10012345" }`
  - Signed-in customer: one extra `init` call before the event, with `user: { email_sha256: "<64 hex>", phone_number_sha256: "<64 hex>" }`. The phone hash is of `966500000000` (no `+`). Check it in the console with `debug: true`.

## 7. Not built, and why

| Zid event | Reason |
|---|---|
| `view_cart`, `remove_from_cart`, `add_to_wishlist`, `add_shipping_info`, `add_payment_info` | OpenAI has no standard event for view cart, remove from cart, wishlist, shipping info or payment info. `checkout_started` already covers the checkout flow. |
| Zid's `email_hashed`, `mobile_hashed`, `external_id` | Zid's hashing method could not be matched to OpenAI's normalisation (section 2, fact 7). `external_id_sha256` only matches the store's own server-side events, which don't exist (fact 8). |
| Names, city, country | Zid's dataLayer carries none of them. |

If the user asks for one of these, build it as a `custom` event on the matching Zid trigger:

```js
window.oaiq("measure", "custom", { type: "custom" }, { custom_event_name: "add_to_wishlist" });
```

`custom_event_name`: 1–64 characters, lowercase letters, digits, `_` or `-`, starting and ending with a letter or digit, and not a standard event name.

**Automatic advanced matching.** OpenAI documents an option where the pixel finds customer data on the page and hashes it itself, but not how to switch it on. This build passes the customer data explicitly instead.

**Consent.** Zid stores normally run without a consent banner, so the tags carry no consent checks. If the store has one, call `oaiq("consent", false)` before `init` and `oaiq("consent", true)` after opt-in, and say so in the report.
