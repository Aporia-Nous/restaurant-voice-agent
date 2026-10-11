# Ordering System Integration (vendor neutral)

This explains how the Willow & Wick phone agent connects to a restaurant's own ordering system: Toast, Square, Clover, a Stripe-based checkout, or a custom system. The agent never talks to a vendor directly. It talks to one small **adapter**: a thin web service, ideally a single serverless function, that turns four simple calls into whatever the restaurant's system expects.

```
Caller ⇄ Votel voice agent ──HTTPS JSON──▶ Adapter ──vendor API──▶ Toast / Square / Clover / Stripe / custom
                                           ◀── payment status webhook ──┘
```

**Demo status:** every endpoint below points at `https://pos-adapter.example.invalid/v1`. The `.invalid` domain can never resolve, so nothing can be sent anywhere. The token is `DUMMY_POS_TOKEN_NOT_REAL`. No real orders, payments, or texts are possible.

---

## 1. Where it sits in the call (Votel workflow `trigger_call_inbound`)

| # | Step id | Votel step type | What it does | If it fails |
|---|---------|-----------------|--------------|-------------|
| 1 | `greet` | talking (static) | Greeting | — |
| 2 | `menu_fetch` | **api_lookup GET** `/menu` | Loads the live menu, the sold-out list, and the menu version | `goto host`: the agent uses the knowledge-base menu and tells the caller staff will confirm prices |
| 3 | `host` | talking (AI) | Takes the order and checks each item and modifier against the live menu | — |
| 4 | `extract_order` | extract_dynamic_variables | Turns the conversation into structured cart fields (items JSON, name, phone, pickup or delivery, etc.) | — |
| 5 | `calc_cents` → `calc_round` → `calc_totals` | set_and_format_variables ×3 | Works out the subtotal, 6.5% tax rounded to the cent, the fee, and the total **by formula, not by the AI** | — |
| 6 | `cart_validate` | **api_lookup POST** `/cart/validate` | The ordering system re-prices and checks the whole cart | `goto readback` using the formula totals plus "staff will confirm the final price" |
| 7 | `readback` | talking (AI), routes | Reads back the full order with exact figures | Change the order → `goto host` |
| 8 | `order_submit` | **api_lookup POST** `/orders` | Sends the confirmed order | **error branch:** `fail_summary` → `notify_staff` (in-app notification to DeAli Dillard) → "staff will confirm shortly" → summary → end |
| 9 | `pay_choice` | talking (AI), routes | Says "your order is in" only on success, then offers a text link or paying at pickup or the door | — |
| 10 | `pay_link` | **api_lookup POST** `/payments/links` | Creates a secure payment link | `link_fail`: pay at pickup or the door |
| 11 | `sms_link` | sms | Texts the link (with the caller's consent) | `error_handling: continue`; the spoken line already says "if it doesn't come through, pay at pickup" |
| — | separate workflow `trigger_webhook` | webhook trigger → send_internal_notification | Gets paid / failed / expired payment updates | — |

Why `api_lookup` and not the `webhook` step: in Votel, the `webhook` step has **no success or failure branches** (it only fires and moves on). `api_lookup` has `success` and `error` branches and maps response fields into variables, so every call to the ordering system uses `api_lookup`, POST included.

**Adapter rule:** return HTTP **2xx only when the action truly succeeded**. Use 4xx or 5xx for "rejected", "item sold out", "store closed", and so on. Votel branches on the HTTP status, so a `200` with `"status":"rejected"` would be treated as success.

Every request carries:
```
Authorization: Bearer <adapter token>      (placeholder today: DUMMY_POS_TOKEN_NOT_REAL)
Content-Type: application/json
X-Demo-Mode: true                          (remove for production)
Idempotency-Key: ww-<call id>              (orders only; protects against double submission)
```

Timeouts are set to 4 seconds (menu), 5 seconds (validate, payment link), and 8 seconds (orders), with no retries during the call. The Votel docs don't say whether `timeout` is in seconds or milliseconds; confirm with the step's Test button before going live.

---

## 2. Menu lookup — `GET /menu?location_id=…&channel=phone`

### Request
Query parameters: `location_id` (string), `channel` = `phone`.

### Response (200)
```json
{
  "menu_version": "mv-2026-10-08-1700",
  "menu_summary": "id H-BURG Wick Burger 15.00, options: cheddar +1.50 (M-CHED), bacon +2.00 (M-BACN), avocado +2.00 (M-AVO), temp rare/medium/well; id H-TACO Fish Tacos (2) 15.00, options: grilled or beer-battered, corn tortillas, no crema - no other add-ons; ...",
  "unavailable_summary": "M-TROU Market Catch Trout - SOLD OUT tonight; D-LEMN Lemonade - unavailable",
  "items": [
    {
      "item_id": "H-BURG",
      "name": "Wick Burger",
      "category": "Handhelds",
      "price": 15.00,
      "available": true,
      "allergens": ["gluten", "dairy", "egg", "sesame"],
      "modifier_groups": [
        {"group_id": "G-ADD", "name": "Add-ons", "min": 0, "max": 3,
         "options": [
           {"modifier_id": "M-CHED", "name": "Cheddar", "price": 1.50, "available": true},
           {"modifier_id": "M-BACN", "name": "Bacon", "price": 2.00, "available": true},
           {"modifier_id": "M-AVO",  "name": "Avocado", "price": 2.00, "available": true}]},
        {"group_id": "G-TEMP", "name": "Temperature", "min": 1, "max": 1,
         "options": [{"modifier_id": "T-RARE", "name": "Rare", "price": 0}, {"modifier_id": "T-MED", "name": "Medium", "price": 0}, {"modifier_id": "T-WELL", "name": "Well", "price": 0}]}
      ]
    }
  ]
}
```
**Votel mapping:** `menu_summary`, `unavailable_summary`, `menu_version`. Votel maps response fields into *text* variables, so the adapter must provide **`menu_summary`**: a compact plain-text version of `items` (ids, names, prices, allowed options) that the voice agent reads. Keep it under about 6,000 characters. Leave out items and options that are unavailable, or list them in `unavailable_summary`. `items` is the structured version, for staff tools and logging.

How the agent uses it (prompt rules, all tested):
- (a) The item must be on the live menu. (b) It must not be in the unavailable list. (c) The agent uses the live price. (d) Only listed modifiers are accepted.
- If an item is missing or sold out, the agent offers the one or two closest available alternatives.
- If an option isn't allowed, the agent says so and offers the allowed options.
- If the menu is blank or the lookup failed, the agent uses the knowledge-base menu and says once that "final prices will be confirmed by staff".

## 3. Cart validation — `POST /cart/validate`

### Request
```json
{
  "schema": "cart-validate/v1",
  "location_id": "willow-wick-demo",
  "menu_version": "mv-2026-10-08-1700",
  "fulfillment": "pickup",
  "items": [
    {"name": "Wick Burger", "item_id": "H-BURG", "quantity": 1, "unit_price": 15.00,
     "modifiers": [{"name": "Cheddar", "modifier_id": "M-CHED", "price": 1.50}],
     "special_instructions": "cooked medium"}
  ],
  "subtotal_estimate": 16.5
}
```
### Response (200 means the check ran; the result is in the body)
```json
{
  "valid": true,
  "problems_summary": "",
  "problems": [],
  "subtotal": "16.50",
  "tax": "1.07",
  "delivery_fee": "0.00",
  "total": "17.57",
  "ready_estimate_text": "about 25 minutes",
  "cart_token": "ct_9f2a…",
  "resolved_items": [{"item_id": "H-BURG", "modifier_ids": ["M-CHED"], "line_total": "16.50"}]
}
```
When there are problems, return `"valid": false` plus a speakable `problems_summary`, for example `"Lemonade is unavailable right now; Wick Burger is 15.50 tonight"`, and a structured `problems` list: `[{"item_id":"D-LEMN","code":"unavailable"},{"modifier_id":"M-BACN","item_id":"H-TACO","code":"modifier_not_allowed"},{"item_id":"H-BURG","code":"price_changed","old":15.00,"new":15.50}]`.

The adapter should match by `item_id` and `modifier_id` when they're present, and fall back to a forgiving name match when they're blank (when the menu lookup failed, the agent only knows names).

**Votel mapping:** `valid`, `problems_summary`→`problems`, `subtotal`, `tax`, `delivery_fee`, `total`, `ready_estimate_text`→`ready`, `cart_token`.

## 4. Order submission — `POST /orders`

### Order payload schema (`phone-order/v1`)
| Field | Type | Notes |
|-------|------|-------|
| `schema` | string | `"phone-order/v1"` |
| `location_id` | string | The restaurant or location key in the adapter |
| `source` | string | `"voice_agent"` |
| `call_id` | string | Votel call id (`{{call.id}}`), for tracing |
| `client_order_ref` | string | `ww-<call id>`. Use it as the idempotency key and as the vendor's external reference |
| `menu_version` | string | From the menu lookup (blank if the lookup failed) |
| `cart_token` | string | From validation (blank if validation failed) |
| `customer.name` / `customer.phone` / `customer.caller_id` | string | Phone as spoken, plus the caller ID |
| `fulfillment.type` | `"pickup"` \| `"delivery"` | |
| `fulfillment.requested_time` | string | `"ASAP"` or the spoken time |
| `fulfillment.delivery_address` | string | Blank for pickup |
| `items[]` | array | `name`, `item_id`, `quantity`, `unit_price`, `modifiers[] {name, modifier_id, price}`, `special_instructions` |
| `special_instructions` | string | Order level |
| `allergy_notes` | string | Always print these on the kitchen ticket |
| `totals_estimate` | object | `currency`, `subtotal`, `tax_rate` (0.065), `tax`, `delivery_fee`, `total`. **An estimate only: the ordering system's own pricing is the source of truth** |
| `payment` | object | `{"status":"unpaid","method":"pending"}` at submission |

### Example payload
```json
{
  "schema": "phone-order/v1",
  "location_id": "willow-wick-demo",
  "source": "voice_agent",
  "call_id": "c_01HZX…",
  "client_order_ref": "ww-c_01HZX…",
  "menu_version": "mv-2026-10-08-1700",
  "cart_token": "ct_9f2a…",
  "customer": {"name": "Casey", "phone": "+16055550199", "caller_id": "+16055550199"},
  "fulfillment": {"type": "delivery", "requested_time": "ASAP", "delivery_address": "220 Oak Avenue, Millbrook, SD"},
  "items": [
    {"name": "Fish Tacos (2)", "item_id": "H-TACO", "quantity": 1, "unit_price": 15.00,
     "modifiers": [{"name": "Beer-battered", "modifier_id": "M-BATR", "price": 0},
                   {"name": "Corn tortillas", "modifier_id": "M-CORN", "price": 0}],
     "special_instructions": "one taco with no crema"},
    {"name": "Lemonade", "item_id": "D-LEMN", "quantity": 1, "unit_price": 4.00, "modifiers": [], "special_instructions": ""}
  ],
  "special_instructions": "",
  "allergy_notes": "",
  "totals_estimate": {"currency": "USD", "subtotal": 19.0, "tax_rate": 0.065, "tax": 1.24, "delivery_fee": 4.99, "total": 25.23},
  "payment": {"status": "unpaid", "method": "pending"}
}
```
### Response
`201` (or `200`):
```json
{"order_id": "W-1001", "status": "accepted", "ready_time_text": "about 25 minutes", "checkout_url": null}
```
Any non-2xx status sends the call to the staff fallback. **Votel mapping:** `order_id`→`pos_order_id`, `status`→`order_status`, `ready_time_text`→`ready_time`.

The agent says the order is "in" **only** on the success branch. On failure it says: "I have passed your order to our team, and someone will confirm it with you shortly. Payment is at pickup or at the door." Staff get an in-app Votel notification containing a ready-to-enter ticket. It can also go out by text if DeAli's notification settings send workflow notifications by SMS. A live transfer to +1 555-555-0124 stays available any time the caller asks for a person.

---

## 5. Payments (compliant, vendor neutral)

### Rules the agent follows (in the system prompt, the `host` step, and the `pay_choice` step; all tested)
- It **never** collects card numbers, expiry dates, CVV codes, or bank details by voice, and never repeats digits a caller says. This keeps card data out of PCI DSS scope for the voice channel.
- A caller who starts reading a card number is stopped politely: "For your security, please don't read card details over the phone. I can text you a secure payment link instead, or you can pay at pickup."
- Payment is only offered **after** the ordering system accepts the order. There are two choices: a **text link now**, or **pay at pickup or the door**.
- **Consent to text (TCPA):** before texting, the agent asks "Is it okay if I text a secure payment link to the number you're calling from?" It routes to the text-link path only after a clear yes. The text identifies the business and includes "Reply STOP to opt out". It's a one-off transactional message tied to the caller's own request, never marketing.

### Create payment link — `POST /payments/links`
Request (`payment-link/v1`):
```json
{
  "schema": "payment-link/v1",
  "location_id": "willow-wick-demo",
  "order_id": "W-1001",
  "client_order_ref": "ww-c_01HZX…",
  "amount_estimate": 17.57,
  "currency": "USD",
  "customer_phone": "+16055550101",
  "expires_in_minutes": 30,
  "description": "Willow & Wick Kitchen phone order"
}
```
Response (`201`):
```json
{
  "payment_url": "https://pay.example-restaurant.com/l/abc123",
  "payment_link_id": "pl_abc123",
  "expires_at": "2026-10-08T18:45:00-05:00",
  "kind": "payment_link"
}
```
- The adapter must charge the **ordering system's own order total** for `order_id`. `amount_estimate` is only a cross-check, and the adapter should refuse (non-2xx) if the two differ by more than a few cents.
- If the restaurant's system can create a **full online order with checkout** (an online-ordering page that already holds the cart), return that URL as `payment_url` with `"kind": "checkout"`. The agent and the text message stay the same.
- **Votel mapping:** `payment_url`, `expires_at`, `payment_link_id`.
- Failure (non-2xx) → the agent says it couldn't create a link, so the caller pays at pickup or the door.

### Payment status webhook (adapter → Votel workflow `trigger_webhook`)
Votel creates the webhook URL when the agent is activated (it doesn't exist yet). The adapter POSTs:
```json
{
  "event": "payment.updated",
  "status": "paid",
  "order_id": "W-1001",
  "client_order_ref": "ww-c_01HZX…",
  "payment_link_id": "pl_abc123",
  "amount": "17.57",
  "currency": "USD",
  "occurred_at": "2026-10-08T18:21:07-05:00",
  "signature": "<HMAC of body with shared secret>"
}
```
`status` is `paid`, `failed`, or `expired`. The Votel workflow sends DeAli Dillard an in-app notification: "Phone order W-1001: payment paid", with "If failed or expired, collect payment at pickup or the door". The adapter should also mark the order paid or unpaid in the ordering system itself, which most vendors do automatically for their own links.

### Compliance checklist
1. **No card data in transcripts or recordings.** The agent never asks for card data and stops anyone who starts reading it. A caller can still say digits before being stopped, though, so before going live: turn on transcript and recording redaction of card numbers if Votel offers it, or limit recording and transcript retention and access. Do this alongside the text-link flow, not instead of it.
2. **The link expires** (30 minutes by default; `expires_in_minutes`). Expired links post `expired` to the webhook, and staff collect at pickup.
3. **Hosted payment pages only.** Card entry happens on the vendor's PCI-compliant hosted page. The adapter and Votel never see card numbers.
4. **TCPA consent** is spoken and logged in the transcript and the call summary. Texts are transactional, identify the business, and include STOP opt-out.
5. **SMS sender:** +1 555-555-0123 is **not SMS-registered** (A2P 10DLC is needed). Votel reports the channel as "unavailable: no SMS plan is assigned". The `sms_link` step is configured but **cannot send** until the number is registered and given an SMS plan. Until then, the agent's spoken line already falls back to "if it doesn't come through, pay at pickup".
6. **Webhook security:** check an HMAC signature or a shared secret on the webhook, and use HTTPS only.

---

## 6. Adapter notes per vendor (mapping only, no lock-in)

| Our call | Toast | Square | Clover | Stripe (Payment Links / Checkout) | Custom system |
|----------|-------|--------|--------|-------------------------------|---------------|
| `GET /menu` | Menus API (menu groups, items, modifier groups) plus stock/availability for 86'd items | Catalog API (ITEM, ITEM_VARIATION, MODIFIER_LIST) plus Inventory counts or "sold out" flags | Inventory API (items, modifier groups, item stock / `available` flag) | Not a menu system: keep the menu in the adapter or a CMS | Your menu table → `menu_summary` text plus `items` |
| `POST /cart/validate` | Orders API price check (or rebuild the order and price it without sending it) | `CalculateOrder` returns line totals and taxes without creating anything | Create an order in a "draft" state, or recompute from Inventory prices | Price from the adapter's own menu | Your pricing engine |
| `POST /orders` | Orders API: create an order with `externalReferenceId = client_order_ref` | `CreateOrder` with `idempotency_key`, `reference_id`, and pickup/delivery `fulfillments` | Orders API: create order → add line items and modifications → print or fire | No order system: the adapter writes to a ticket printer, KDS, or sheet, then uses Stripe only for payment | Insert into your orders table and push to KDS |
| `POST /payments/links` | Toast online ordering / payment link (if enabled), else a Stripe link referencing the Toast order | Checkout API `CreatePaymentLink` for the existing `order_id` | Hosted Checkout / payment link for the order | Payment Link or Checkout Session with `metadata.order_id`, `expires_at` | Your hosted checkout URL |
| Payment webhook | Toast webhooks (payment / order updated) → normalize → POST to Votel | `payment.updated` / `order.updated` webhooks → normalize | Clover webhooks (payments) → normalize | `checkout.session.completed` / `.expired`, `payment_intent.payment_failed` → normalize | Your payment events → normalize |

Rules every adapter follows:
- Translate errors into non-2xx statuses.
- Prices are always recomputed on the restaurant side.
- Idempotency via `client_order_ref`.
- Every phone order is tagged with `source = phone/voice agent` so staff can filter.
- Allergy notes always print on the kitchen ticket.

Vendor API details change. Check each vendor's current docs when building a real adapter.

---

## 7. Known limits (what Votel allows today)
- **Fixed points, not free-form tool calls.** In a scripted workflow the menu is loaded **once** at the start of the call (`menu_fetch`) and the cart is checked **once** before the read-back (`cart_validate`). Item-by-item checks during the conversation use that loaded menu. Votel's "inline tools" (where the AI calls an API whenever it likes) are only for prompt-only agents without a workflow, so they aren't used here.
- **`webhook` step can't branch** on success or failure. That's why `api_lookup` is used everywhere.
- **Mapped response values are text.** That's why the adapter returns a speakable `menu_summary` and `problems_summary`.
- **JSON bodies are text templates.** `items` is inserted raw from the AI-extracted `items_json`. If that's malformed, or a name contains an unescaped quote, the request fails and the call takes the **safe staff-fallback path**. The adapter should also accept and repair loosely formatted items.
- **Tax math** is done by formula in Votel, not by the AI: cents are rounded half-up using a split-on-decimal trick, because Votel's math has no `round()`. The ordering system's figures replace it whenever validation succeeds.
- **`{{call.id}}` / `{{call.caller_id_number}}`** pass Votel's checks, but they're blank in dry runs. Confirm they fill in on the first live test call.
- **The text-message step has no failure branch.** It's set to continue on error, and the spoken line covers a missed text.
