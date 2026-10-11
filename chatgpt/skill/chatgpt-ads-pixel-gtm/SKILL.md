---
name: chatgpt-ads-pixel-gtm
description: Use when asked to install, set up or audit the ChatGPT Ads / OpenAI measurement pixel (oaiq) or ChatGPT conversion events through Google Tag Manager on a custom-built website (not Salla, Zid or Shopify), using a GTM MCP server and a browser.
---

# ChatGPT Ads (OpenAI) Pixel via GTM — Custom Websites

## Overview

A custom website has no known dataLayer. So the build is derived, in this order, from three things you **observe** — never assume: (1) OpenAI's pixel docs, (2) the business model, (3) the site's real dataLayer and the container's existing triggers/variables.

Pixel facts, event catalogue and tag code live in `reference.md` (the REFERENCE section below if this is a pasted prompt). Read it before Step 3.

## Step 0 — Inputs gate (do not start without all three)

```
Website URL ............ <https://...>
GTM account + container  <account name> / <container name>
OpenAI Pixel ID ........ <from Ads Manager → Conversions>
```

If any is missing: ask for it and **stop**. No browsing, no GTM reads.

Resolve the container by listing accounts, then containers in that account. Names given by people are approximate ("acme store under agency new" → account `Agency - new`, container `Acme - Store`). One clear match → state it and continue. Zero or several plausible matches → ask. Confirm the match by checking the container's public ID (`GTM-XXXX`) is the one loaded on the website.

## Step 1 — Load the reference

Read `reference.md`. If you have web access, also open `https://developers.openai.com/ads/measurement-pixel` and `https://developers.openai.com/ads/supported-events`; where the live docs differ from the reference, the live docs win — say what changed.

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
