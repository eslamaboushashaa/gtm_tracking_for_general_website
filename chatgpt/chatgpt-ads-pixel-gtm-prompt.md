# ChatGPT Ads (OpenAI) Pixel via GTM — Any Custom Website — Reusable Prompt

Paste everything below the line into any LLM agent that has a **GTM MCP server** and a **browser** connected.
Fill in the three INPUTS in Step 0. Nothing else changes between clients or industries.
(Claude Code users: install the skill in `skill/chatgpt-ads-pixel-gtm/` instead — same workflow.)

---

You are setting up the ChatGPT Ads (OpenAI) measurement pixel on a custom-built website through Google Tag Manager. The website can be any kind of business; you work out which kind in Step 2. Follow the workflow below step by step, in order. "The reference" means the REFERENCE section at the end of this prompt.


## WORKFLOW

## Overview

A custom website has no known dataLayer. So the build is derived, in this order, from three things you **observe** — never assume: (1) OpenAI's pixel docs, (2) the business model, (3) the site's real dataLayer and the container's existing triggers/variables.

Pixel facts, event catalogue and tag code live in the REFERENCE section at the end of this prompt. Read it before Step 3.

## Step 0 — Inputs gate (do not start without all three)

```
Website URL ............ <https://...>
GTM account + container  <account name> / <container name>
OpenAI Pixel ID ........ <from Ads Manager → Conversions>
```

If any is missing: ask for it and **stop**. No browsing, no GTM reads.

Resolve the container by listing accounts, then containers in that account. Names given by people are approximate ("acme store under agency new" → account `Agency - new`, container `Acme - Store`). One clear match → state it and continue. Zero or several plausible matches → ask. Confirm the match by checking the container's public ID (`GTM-XXXX`) is the one loaded on the website.

## Step 1 — Load the reference

Read the REFERENCE section. If you have web access, also open `https://developers.openai.com/ads/measurement-pixel` and `https://developers.openai.com/ads/supported-events`; where the live docs differ from the reference, the live docs win — say what changed.

## Step 2 — Understand the business

Open the website and browse it like a customer. Classify what a conversion **is** for this business:

| Business type | Signals on the site | Primary conversion |
|---|---|---|
| Purchase (any online payment: store, tickets, courses, orders) | catalogue, cart, checkout, payment | `order_created` |
| Lead generation (services, real estate, B2B, clinics, agencies) | contact/quote forms, WhatsApp/call buttons, no checkout | `lead_created` |
| Subscription / SaaS | pricing plans, sign-up, trial | `subscription_created`, `trial_started`, `registration_completed` |
| Booking | calendar, demo/consultation/appointment booking | `appointment_scheduled` |
| Content / community / other | articles, membership, downloads, no payment | `registration_completed`, or `custom` |

A site can be more than one type (a store with a "request a quote" form is purchase + lead). State the type and one line of evidence. If the site does not make its conversion obvious, ask the user what action they want to optimise ads for — one question, then continue.

## Step 3 — Decide the events

Pick events from the per-business table in the reference. `page_viewed` is always built. Use a standard event whenever one fits; `custom` only when none does. Keep the list — every event now needs a data source (Step 4) and a trigger (Step 5).

## Step 4 — Read the real dataLayer

In the browser, on the live site, run this in the page (console equivalent of typing `dataLayer`):

```js
JSON.stringify((window.dataLayer || []).filter(function (e) { return e && e.event && !/^gtm\./.test(e.event); }))
```

Walk the funnel for the business type and re-run it after each action:

- Purchase: home → listing → item page → add to cart → cart → start checkout.
- Lead: home → service page → open the form (and click WhatsApp/call buttons if they are lead channels).
- Subscription / booking: home → pricing or services → sign-up / booking page, up to the last step before submitting.

Rules while walking:
- Adding to cart and opening checkout are fine. **Never** pay, submit a real form, create an account, or type personal data.
- For events you cannot trigger (purchase, form success): find the push shape in the site's own JS — fetch the same-origin `<script src>` bundles and search for the event name (`"purchase"`, `transaction_id`, `generate_lead`). Cross-check with the container's existing purchase tags and variables.
- Also record: which GTM container ID is on the page, whether navigation is SPA (no full reload between pages), and which pixels already run.

Write a small table: OpenAI event → dataLayer event name → exact paths for value, currency, items, order ID, user email/phone → verified live or inferred from code.

If the dataLayer has **no** usable event for a needed step, fall back in this order: URL-based Page View trigger (thank-you page) → Form Submission / Click trigger → DOM scraping variable. Tell the user a developer push would be more reliable.

## Step 5 — Inventory the container, reuse first

List workspaces, triggers, variables, folders and tags (tag lists are huge — save and grep for `"name"`, `"type"`, `firingTriggerId`).

- **Trigger exists for the event → reuse its ID. Do not create a duplicate.** Read its filters: an existing trigger may be deliberately narrower (e.g. `purchase` excluding unpaid/offline payment methods, `add_to_cart` excluding upsells). Follow the precedent the other ad-platform tags use; if a tag needs the union, attach several existing triggers.
- **Variable exists for a path** (value, currency, items, transaction ID) → reference it by its existing name.
- Create a trigger only when nothing matches. Never edit, rename or delete anything that was already there.

## Step 6 — Build (GTM MCP)

1. Create a dedicated workspace `EA - OpenAI Pixel`. Never build in the default workspace.
2. One `bulkUpdate` with temporary IDs (`new_1`, …):
   - Folder `EA - OpenAI` — everything new goes in it.
   - Constant variable `EA - OpenAI - pixel_id` = the Pixel ID. **The ID appears nowhere else.**
   - `EA - OpenAI - amount` and `EA - OpenAI - contents` custom JS variables (reference templates), pointed at the paths found in Step 4.
   - Custom HTML tags from the reference templates: `EA - OpenAI - init` plus one tag per event.

Naming — always `EA - <Platform> - <event>`, with the platform's own event name:

| Entity | Name |
|---|---|
| Tags | `EA - OpenAI - init`, `EA - OpenAI - page_viewed`, `EA - OpenAI - order_created`, … |
| Variables | `EA - OpenAI - pixel_id`, `EA - OpenAI - amount`, `EA - OpenAI - contents` |
| Folder / workspace | `EA - OpenAI` / `EA - OpenAI Pixel` |
| New triggers (only if none existed) | `EA - Trigger - <dataLayer event name>` |

Put a `notes` line on every entity saying what it fires on and why.

## Step 7 — Verify, then stop

1. `quickPreview` the workspace; search the result for `compilerError`.
2. In the browser, run the amount/contents mapping against a real captured push and check the output against the reference rules (integers, minor units, allowed fields only).
3. **Do not create a version. Do not publish.** Publishing is the user's call, every time.

## Step 8 — Report

Business type · events built (OpenAI event ← dataLayer event ← trigger, reused or new) · variables reused vs created · events deliberately not built and why · what was inferred rather than seen live · the user's Preview checklist (one test per tag, incl. one real test conversion) · "nothing is published".

Never leave `debug: true` in a tag you hand over. Production `init` carries `pixelId` (and `user` when available) only.

## Common mistakes

| Mistake | Fix |
|---|---|
| Sending `amount: 129.99` for $129.99 | Integer **minor units**: `12999`. Use the amount variable. |
| `debug: true` left in `init` | Remove it before handing over. |
| `amount` without `currency` | Currency is required whenever amount is sent. |
| Email/phone inside `measure` | User data goes in `init` only, SHA-256 hashed, never raw. |
| Extra fields in `contents[]` (category, brand, url) | Only `id`, `name`, `content_type`, `quantity`, `amount`, `currency`. |
| `"{{EA - OpenAI - pixel_id}}"` in quotes inside Custom HTML | GTM inserts the quoted literal itself: write `{{EA - OpenAI - pixel_id}}` bare. |
| ES6 in GTM (`const`, arrow functions, template strings) | ES5 only in Custom HTML and Custom JS variables. |
| Two dataLayer sources feeding one event (`purchase` + `order_placed`) | One source per funnel step. |
| New trigger that duplicates an existing one | Reuse the existing trigger ID. |
| `page_viewed` only on page load on an SPA | Add the History Change trigger with a pathname guard. |
| Purchase re-counted on thank-you reload | Pass `event_id` = order/transaction ID. |
| Publishing because "it compiled" | Stop at the workspace. Ask. |

---

# REFERENCE — OpenAI (ChatGPT Ads) measurement pixel

Source: `developers.openai.com/ads/measurement-pixel` and `/ads/supported-events`, read 2026-10-11. There is no official GTM template — every tag is Custom HTML.

## 1. Pixel API

**Loader + init** (official snippet):

```html
<script>
  (function (w, d, s, u) {
    if (w.oaiq) return;
    var q = function () {
      q.q.push(arguments);
    };
    q.q = [];
    w.oaiq = q;
    var js = d.createElement(s);
    js.async = true;
    js.src = u;
    var f = d.getElementsByTagName(s)[0];
    f.parentNode.insertBefore(js, f);
  })(window, document, "script", "https://bzrcdn.openai.com/sdk/oaiq.min.js");
  oaiq("init", {
    pixelId: "<YOUR-PIXEL-ID>",
  });
</script>
```

- `oaiq("init", { pixelId, user })` — `pixelId` required. A `debug: true` option exists that logs SDK activity to the console; use it only while testing by hand and never ship it in a tag.
- `oaiq("measure", eventName, eventData, options)` — `eventData.type` must match the event. `options`: `event_id` (dedup), `custom_event_name` (custom events only), `opt_out`.
- `oaiq("consent", false)` **before** `init` when consent is required; `oaiq("consent", true)` after opt-in. Consent defaults to granted. Blocked events are not replayed.
- Dedup key = pixel ID + event name + `event_id`. First event wins. Reuse the same `event_id` in browser and Conversions API.
- Automatic: captures the `oppref` click parameter into the `__oppref` cookie (30 days), sets `__obref` (365 days), adds `source_url` and timestamp, batches calls.
- CSP if the site has one: `script-src https://bzrcdn.openai.com`; `connect-src https://bzr.openai.com https://bzrcdn.openai.com`; `img-src https://bzr.openai.com`.
- `app_installed` / `app_opened` are Conversions API only — never from the pixel.

## 2. Events and data shapes

| Event | `type` | Meaning |
|---|---|---|
| `page_viewed` | `contents` | Page load / important page view |
| `contents_viewed` | `contents` | Product, listing, article, project viewed |
| `items_added` | `contents` | Added to cart / bundle / selection |
| `checkout_started` | `contents` | Checkout begins |
| `order_created` | `contents` | Purchase completed |
| `lead_created` | `customer_action` | Lead form submitted / contact requested |
| `registration_completed` | `customer_action` | Account or event registration finished |
| `appointment_scheduled` | `customer_action` | Meeting, demo or consultation booked |
| `subscription_created` | `plan_enrollment` | Paid subscription starts |
| `trial_started` | `plan_enrollment` | Free trial starts |
| `custom` | `custom` | Anything else; needs `custom_event_name` |

Fields per `type`:

| `type` | Fields (all optional except `type`) |
|---|---|
| `contents` | `amount`, `currency`, `contents[]` |
| `customer_action` | `amount`, `currency` |
| `plan_enrollment` | `plan_id`, `amount`, `currency`, `contents[]` |
| `custom` | `plan_id`, `amount`, `currency`, `contents[]` |

`contents[]` item — **only these fields from the pixel**: `id` (string), `name`, `content_type` (`product`, `plan`, `page`, …), `quantity` (integer), `amount` (per-unit, integer minor units), `currency`.

**Amount rules**
- Integer in the currency's ISO 4217 minor unit: 129.99 USD → `12999`; 100 SAR → `10000`; 5 KWD → `5000` (3 decimals); 500 JPY → `500` (0 decimals).
- Event `amount` = value of the whole event. Item `amount` = price of **one** unit; send `quantity` separately.
- `amount` present → `currency` required.

**Custom event names**: 1–64 chars, lowercase letters, digits, `_` or `-`, start and end alphanumeric, not a standard event name.

**User data** — `init` only, never `measure`. All SHA-256, 64-char lowercase hex, never raw:

| Field | Normalise before hashing |
|---|---|
| `email_sha256` | trim, lowercase |
| `phone_number_sha256` | digits with country code; strip spaces, `()`, `.`, `-`, leading `+` and leading zeros; 8–15 digits |
| `external_id_sha256` | trim only, keep case |
| `first_name_sha256`, `last_name_sha256` | lowercase, remove whitespace and ASCII punctuation, keep non-ASCII |
| `country` (ISO-2), `city`, `region`, `postal_code` | plain, not hashed |

To update user data later (after login / on the order page) call `init` again with the full `user` object.

## 3. Events to build per business type

| Business type | Build | Typical dataLayer source |
|---|---|---|
| Purchase (any online payment) | `page_viewed`, `contents_viewed`, `items_added`, `checkout_started`, `order_created` | `view_item`, `add_to_cart`, `begin_checkout`, `purchase` |
| Lead generation | `page_viewed`, `contents_viewed` (service pages, optional), `lead_created`; `custom` (`whatsapp_click`, `phone_click`) if those are real lead channels | `generate_lead`, form-success push, thank-you URL |
| Subscription / SaaS | `page_viewed`, `contents_viewed` (pricing), `registration_completed`, `trial_started`, `subscription_created` | `sign_up`, `start_trial`, `purchase`/`subscribe` |
| Booking | `page_viewed`, `lead_created` (enquiry), `appointment_scheduled` | booking-confirmed push or URL |
| Content / community / other | `page_viewed`, `contents_viewed`, `registration_completed`, or one `custom` event for the action the user cares about | `sign_up`, download/click push, URL |

No OpenAI standard event exists for: view cart, remove from cart, search, wishlist, login, payment-info, checkout errors. Do not build them unless the user asks for a `custom` event.

## 4. GTM templates (ES5 only)

`{{…}}` names below are placeholders for the container's real variables — reuse existing ones (value, currency, items, transaction ID) where they exist.

### 4.1 `EA - OpenAI - init` — Custom HTML

Trigger: **Initialization - All Pages** (built-in ID `2147479573`). Firing option: **once per page**.

```html
<script>
  (function (w, d, s, u) {
    if (w.oaiq) return;
    var q = function () {
      q.q.push(arguments);
    };
    q.q = [];
    w.oaiq = q;
    var js = d.createElement(s);
    js.async = true;
    js.src = u;
    var f = d.getElementsByTagName(s)[0];
    f.parentNode.insertBefore(js, f);
  })(window, document, "script", "https://bzrcdn.openai.com/sdk/oaiq.min.js");
  if (!window.__eaOaiInit) {
    window.__eaOaiInit = true;
    oaiq("init", {
      pixelId: {{EA - OpenAI - pixel_id}}
    });
  }
</script>
```

Every other tag gets this tag as its **setup tag** (`stopOnSetupFailure: false`).

### 4.2 `EA - OpenAI - page_viewed` — Custom HTML

Triggers: **All Pages** (`2147479553`) + the container's History Change trigger when the site is an SPA. The pathname guard stops double fires.

```html
<script>
(function () {
  if (typeof window.oaiq !== 'function') return;
  var path = window.location.pathname;
  if (window.__eaOaiLastPath === path) return;
  window.__eaOaiLastPath = path;
  window.oaiq('measure', 'page_viewed', {
    type: 'contents',
    contents: [{ id: path, content_type: 'page' }]
  });
})();
</script>
```

### 4.3 `EA - OpenAI - amount` — Custom JavaScript variable

```js
function() {
  var n = parseFloat({{value}});
  if (isNaN(n)) { return undefined; }
  var c = String({{currency}} || 'USD').toUpperCase();
  var zero = {JPY:1, KRW:1, VND:1, CLP:1, ISK:1, UGX:1, XAF:1, XOF:1, PYG:1, RWF:1};
  var three = {KWD:1, BHD:1, OMR:1, JOD:1, TND:1, IQD:1, LYD:1};
  var exp = three[c] ? 3 : (zero[c] ? 0 : 2);
  return Math.round(n * Math.pow(10, exp));
}
```

`'USD'` is only a placeholder fallback: replace it, in every template, with the currency the site actually charges in. If the dataLayer has no currency field at all, that fallback becomes the source of truth — say so in the report.

### 4.4 `EA - OpenAI - contents` — Custom JavaScript variable

Written for GA4-style items (`item_id`, `item_name`, `price`, `quantity`). Change the four property names to whatever the site pushes.

```js
function() {
  var items = {{DLV - items}};
  if (!items || !items.length) { return undefined; }
  var c = String({{currency}} || 'USD').toUpperCase();
  var zero = {JPY:1, KRW:1, VND:1, CLP:1, ISK:1, UGX:1, XAF:1, XOF:1, PYG:1, RWF:1};
  var three = {KWD:1, BHD:1, OMR:1, JOD:1, TND:1, IQD:1, LYD:1};
  var mult = Math.pow(10, three[c] ? 3 : (zero[c] ? 0 : 2));
  var out = [];
  for (var i = 0; i < items.length; i++) {
    var it = items[i] || {};
    if (it.item_id === undefined || it.item_id === null) { continue; }
    var row = { id: String(it.item_id), content_type: 'product' };
    if (it.item_name) { row.name = String(it.item_name).replace(/^\s+|\s+$/g, ''); }
    var q = parseInt(it.quantity, 10);
    row.quantity = (isNaN(q) || q < 1) ? 1 : q;
    var p = parseFloat(it.price);
    if (!isNaN(p)) { row.amount = Math.round(p * mult); row.currency = c; }
    out.push(row);
  }
  return out.length ? out : undefined;
}
```

### 4.5 `contents` event tags — Custom HTML

Same body for `contents_viewed`, `items_added`, `checkout_started`; change only the event name.

```html
<script>
(function () {
  if (typeof window.oaiq !== 'function') return;
  var data = { type: 'contents' };
  var amount = {{EA - OpenAI - amount}};
  var contents = {{EA - OpenAI - contents}};
  if (typeof amount === 'number') {
    data.amount = amount;
    data.currency = String({{currency}} || 'USD').toUpperCase();
  }
  if (contents && contents.length) data.contents = contents;
  window.oaiq('measure', 'contents_viewed', data);
})();
</script>
```

`order_created` — identical, but ends with the dedup ID:

```js
  var tid = {{transaction_id}};
  if (tid !== undefined && tid !== null && tid !== '') {
    window.oaiq('measure', 'order_created', data, { event_id: String(tid) });
  } else {
    window.oaiq('measure', 'order_created', data);
  }
```

### 4.6 `customer_action` / `plan_enrollment` / `custom` tags

```html
<script>
(function () {
  if (typeof window.oaiq !== 'function') return;
  window.oaiq('measure', 'lead_created', { type: 'customer_action' });
})();
</script>
```

- Lead with a known value: add `amount` + `currency` exactly as in 4.5.
- Subscription: `{ type: 'plan_enrollment', plan_id: {{plan id}}, amount: …, currency: … }` with event `subscription_created` or `trial_started`.
- Custom: `window.oaiq('measure', 'custom', { type: 'custom' }, { custom_event_name: 'whatsapp_click' });`
- A lead tag that can fire twice for one submission (form event + thank-you page) needs an `event_id` (lead/reference ID) or a single trigger.

### 4.7 User data (only when the dataLayer really exposes email / phone)

Build it as its own tag `EA - OpenAI - user_data` on the trigger where the data becomes available (usually the order or lead event); `init` may be re-called at any time. Already-hashed values that match the normalisation in section 2 can be passed straight through; otherwise hash in the browser:

```html
<script>
(function () {
  if (typeof window.oaiq !== 'function' || !window.crypto || !window.crypto.subtle || !window.TextEncoder) return;
  function sha256(str) {
    return window.crypto.subtle.digest('SHA-256', new TextEncoder().encode(str)).then(function (buf) {
      var b = new Uint8Array(buf), h = '';
      for (var i = 0; i < b.length; i++) { h += ('0' + b[i].toString(16)).slice(-2); }
      return h;
    });
  }
  var email = String({{user_email}} || '').replace(/^\s+|\s+$/g, '').toLowerCase();
  var phone = String({{user_phone}} || '').replace(/[^\d]/g, '').replace(/^0+/, '');
  var jobs = [email ? sha256(email) : null, (phone.length >= 8 && phone.length <= 15) ? sha256(phone) : null];
  Promise.all(jobs).then(function (r) {
    var user = {};
    if (r[0]) user.email_sha256 = r[0];
    if (r[1]) user.phone_number_sha256 = r[1];
    if (user.email_sha256 || user.phone_number_sha256) {
      window.oaiq('init', { pixelId: {{EA - OpenAI - pixel_id}}, user: user });
    }
  });
})();
</script>
```

The phone must include the country code; a local number (`05…`) is not valid — prepend the country code or skip it. If the site only has user data you cannot verify live, leave this tag out and report it.

## 5. What a finished run looks like (illustrative, fictional site)

Use this as the shape of your own notes and final report. Every value comes from what you observed on the real site and container — nothing here is a default.

- **Inputs**: site `https://www.example-shop.com`, account `Agency - new`, container `Example - Shop` (GTM-XXXXXXX, confirmed on the page), pixel ID stored in `EA - OpenAI - pixel_id`.
- **Business type**: purchase — catalogue, cart and card checkout. Also has a "request a quote" form → lead.
- **dataLayer found** (shapes differ on every custom site; record the real one):

```js
{ event: "view_item",  ecommerce: { currency: "USD", value: 129.99, items: [{ item_id: "SKU-1", item_name: "…", price: 129.99, quantity: 1 }] } }
{ event: "add_to_cart", ecommerce: { /* same shape */ } }
{ event: "begin_checkout", ecommerce: { /* same shape */ } }
{ event: "purchase", ecommerce: { transaction_id: "ORD-1001", currency: "USD", value: 259.98, items: [ /* … */ ] } }   // read from the site's JS, not triggered
{ event: "quote_form_submitted" }                                                                                      // seen in the site's JS, not submitted
```

- **Trigger and variable map**:

| Tag | Trigger | Reused or new |
|---|---|---|
| `EA - OpenAI - init` | Initialization - All Pages | built-in |
| `EA - OpenAI - page_viewed` | All Pages (+ History Change if SPA) | built-in / reused |
| `EA - OpenAI - contents_viewed` | existing `view_item` custom event trigger | reused |
| `EA - OpenAI - items_added` | existing `add_to_cart` trigger | reused |
| `EA - OpenAI - checkout_started` | existing `begin_checkout` trigger | reused |
| `EA - OpenAI - order_created` | existing `purchase` trigger (the one the other ad pixels use) | reused |
| `EA - OpenAI - lead_created` | `EA - Trigger - quote_form_submitted` | new — none existed |

- **Variables**: reused the container's value / currency / items / transaction ID variables; created `EA - OpenAI - pixel_id`, `EA - OpenAI - amount`, `EA - OpenAI - contents`.
- **Not built, and why**: view cart, search, login (no OpenAI standard event); user data (no email/phone in any push).
- **Inferred, not seen live**: `purchase` and `quote_form_submitted` shapes.
- **Mapping check** on the live `view_item` push → `{ type: "contents", amount: 12999, currency: "USD", contents: [{ id: "SKU-1", content_type: "product", name: "…", quantity: 1, amount: 12999, currency: "USD" }] }`.
- **State**: workspace `EA - OpenAI Pixel` compiled with no errors; nothing published.
