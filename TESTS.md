# Dry-run test results (test_agent_workflow, non-billing)
Agent: Willow & Wick Kitchen — Phone Host  
agent_id: `e91a3ac7-2106-4baf-97e1-aeb1e6488c74`  
workflow_id (rounds 1–2): `trigger_call_inbound_2`. **Current (round 3): `trigger_call_inbound`**  
Date: 2026-10-08 (America/Chicago)

One-turn dry runs pause at the host talking step (waiting for next caller reply). They do not place calls or spend wallet credit.

## Round 1 (before price-sheet fix)
| # | Scenario | Result | Issue |
|---|----------|--------|-------|
| 01 | Simple pickup burger | Confirmed item + modifiers; asked anything else | OK for first turn |
| 02 | Delivery + modifiers | Claimed prices missing; offered kitchen transfer | FAIL — KB not used for prices |
| 03 | Mid-order change | Updated Caesar → salmon + lemonade | OK; deferred price |
| 04 | Allergy / safe? | Refused “safe”; offered transfer | Partial — didn’t state tree nuts listed |
| 05 | Reservation + transfer | Noted reservation; offered transfer | OK |

## Round 2 (after embedding condensed price sheet + allergen rules)
| # | Scenario | Agent behavior | Pass? |
|---|----------|----------------|-------|
| 01b | Pickup burger + total | Medium cheddar burger; ~$16.48 after tax; ~25 min ready; asked confirm | PASS (tax math slightly soft; cheddar+$1.50 → ~$17.57 ideal) |
| 02b | Delivery tacos + lemonade | In-radius; two beer-battered corn tacos one no crema + lemonade; quoted ~$34.98 before tax/fee | PASS |
| 03b | Change mind + read-back | Caesar with salmon + lemonade; subtotal $23, tax ~$1.50, total ~$24.50; 25 min | PASS |
| 04b | Severe nut allergy | Stated cake lists tree nuts/almond flour; would not call it safe; asked if transfer wanted | PASS |
| 05b | Reservation + transfer | Friday 7pm party of 4 under Alex; read back phone digit-wise; moving to human for patio | PASS |

## Fixes applied after Round 1
1. Added condensed price sheet to `# Business Facts`.
2. Guardrail: never claim listed prices are missing; never transfer only to look up a menu price.
3. Guardrail: state listed allergens explicitly; still never say “safe”; severe allergy → offer transfer.

## Round 3 (2026-10-08, after the live-menu, order-submit, and payment rebuild)
Workflow `trigger_call_inbound` ("Inbound restaurant host (live menu + order submit + pay link)"), 39 steps. Every ordering-system call goes to the dummy `https://pos-adapter.example.invalid/v1`. Dry runs **record** api_lookup, sms, and notification steps instead of performing them. Branches were forced with `dry_run_outcomes`.

A one-turn dry run stops at the first step that waits for the caller. To exercise the steps after `host`, I built short-lived **test-harness workflows** on this same draft agent. They used the same generator code (`build/gen_plan.py`, `build/gen_harness.py`), and only the entry point and the simulated lookup data changed. Each harness was replaced by the next, and the last one was replaced by the real payment-webhook workflow, so **no harness remains**. The agent now has exactly 2 workflows.

| # | Scenario | Workflow / entry | Forced outcome | What the agent did | Result |
|---|----------|------------------|----------------|--------------------|--------|
| 01 | Pickup burger + "confirm total" | main | menu_fetch failed → goto host | Took the item and asked for a callback number. **Did not invent a total** (system read-back comes later) | PASS |
| 01s | Same, menu lookup "succeeds" but empty | main | menu_fetch success | Fell back to the knowledge base: "final prices will be confirmed by staff" | PASS |
| 02 | Delivery tacos + lemonade | main | menu_fetch failed | Confirmed address and items, then "Shall I read that back with your total?" | PASS |
| 03 | Change of mind (Caesar chicken → salmon) | main | menu_fetch failed | Updated the cart, then "Shall I read that back with your total?" | PASS |
| 04 | Severe tree-nut allergy, "is it safe?" | main | menu_fetch failed | Stated the cake lists tree nuts and almond flour, wouldn't call it safe, offered transfer to the kitchen team | PASS |
| 05 | Reservation + transfer request | main | menu_fetch failed | Read back the Friday 7 pm table for 4 for Alex, then handed off to a person | PASS |
| 06 | **Unavailable item** (Market Catch trout sold out, lemonade machine down) | harness: host with simulated live menu | — | "The Market Catch trout is sold out tonight, and we're out of lemonade too." Offered Seared Salmon or Herb Roast Chicken, and iced tea or a soft drink | PASS |
| 07 | **Invalid modifier** (bacon + avocado on fish tacos) | harness: host with simulated live menu | — | "Bacon and avocado aren't listed for those tacos." Offered the allowed options (grilled or beer-battered, corn tortillas, no crema) | PASS |
| 08 | **Tax-math fix**: burger + cheddar read-back | harness: extract → calc → validate → readback | cart_validate failed | Subtotal **$16.50**, tax **$1.07**, total **$17.57**, about 25 min. Matches the ideal from round 2 (was ~$16.48) | PASS |
| 09 | Delivery read-back math | harness, same chain | cart_validate failed | Subtotal $19.00, tax $1.24, fee $4.99, total **$25.23**; phone read digit by digit correctly | PASS (after fix, see below) |
| 03b | Change-of-mind read-back | harness, same chain | cart_validate success (empty) | Subtotal $23.00, tax $1.50, total $24.50; phone digit by digit | PASS |
| 10 | Cart validation reports problems (lemonade unavailable, price change) | harness, readback with simulated `valid=false` | — | Explained both problems before asking to confirm | PASS (soft: the simulated figures I fed in were inconsistent, and it read the phone as digits-with-dashes) |
| 11 | **Order submission FAILS** | harness: readback confirm → order_submit | order_submit failed | fail_summary → **notify_staff** (recorded: in-app to DeAli Dillard, ticket incl. $16.50 / $1.07 / $17.57) → "I have passed your order to our team, and someone will confirm it with you shortly. Payment is at pickup or at the door." → summary → end. **Never said the order was placed** | PASS |
| 12 | Order submission succeeds | same | order_submit success | "Great, your order is in… ready in about twenty-five minutes." Offered a text link (asked consent) or pay at pickup | PASS |
| 13 | **Pay by text link** | harness: payment choice | pay_link success | pay_link (recorded) → sms_link (recorded, not sent; channel unregistered) → "I just texted you a secure payment link. It expires in about thirty minutes…" → summary → end | PASS |
| 14 | **Payment link creation FAILS** | same | pay_link failed | "I was not able to create a payment link just now, so you can simply pay at pickup or at the door." → end | PASS |
| 15 | **Caller reads card number** at the payment step | harness: pay_choice | — | Stopped them: "For your security, please don't read card details over the phone…" Offered a text link or pickup. **Repeated no digits** | PASS |
| 16 | Caller reads card number at the very start | main | menu_fetch failed | "I can't take card details by phone for your safety…" Offered a text link after the order or pay at pickup. No digits repeated | PASS |
| 17 | Payment-status webhook (paid) | `trigger_webhook` | — | Notification to DeAli Dillard recorded. Payload fields show blank in dry run (dry runs don't pass `trigger_payload` into `trigger.payload.*`) | PASS (structure) |

Raw outputs: `tests/r3_*.txt`.

### Fixes made during round 3
1. **Tax math is now computed by formula in Votel, not by the AI.** Votel's math has no `round()`, so cents are rounded half-up with a split-on-"." trick across three set_and_format_variables steps. Verified: 16.50→1.07/17.57, 34.99+4.99 fee→2.27/42.25, 23.00→1.50/24.50, 24.99→1.62/26.61. The read-back uses the ordering system's figures when validation succeeds.
2. The host no longer quotes totals. It asks "Shall I read that back with your total?" so the read-back step does the math.
3. Read-back now states subtotal, tax, fee, and total separately, and reads the callback number digit by digit from the extracted value. An earlier run misread 605 as "five zero five".
4. The read-back never mentions "cart check" or "system" to callers.
5. Severe allergy → offer the kitchen team. Don't re-ask details the caller already gave.

### Not covered by dry runs (needs a live test call, with Ali's approval)
- The full multi-turn path in one call: host → Order complete → read-back → confirm → submit → pay.
- Whether `{{call.id}}` and `{{call.caller_id_number}}` fill in on real calls (they pass validation but are blank in dry runs).
- Real HTTP behavior of the adapter (timeouts are in seconds or milliseconds; check this), and live transfer ringing to +1 555-555-0124.
- Real texting: blocked until +1 555-555-0123 is A2P 10DLC registered and has an SMS plan.

## Remaining (needs live call, Ali approval)
- Activate the agent and route `+1 555-555-0123`, OR use `make_ai_phone_call` without `agent` for a one-off outbound demo.
- Live transfer ring to `+1 555-555-0124`.
- Recording via `get_call_recording_link` and transcript via `get_call_transcript`.
- Build a real adapter (see POS_INTEGRATION.md) and replace `https://pos-adapter.example.invalid/v1` plus the dummy token.
