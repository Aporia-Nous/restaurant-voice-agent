# Willow & Wick Kitchen — Restaurant Voice Agent (DRAFT)

**Status:** Draft only — not activated, no number routed, no live calls placed.  
**agent_id:** `e91a3ac7-2106-4baf-97e1-aeb1e6488c74`  
**Edit URL:** https://app.votel.ai/agents/e91a3ac7-2106-4baf-97e1-aeb1e6488c74/edit  
**Voice:** Katie - Friendly Fixer (Cartesia)  
**Knowledge base:** `8879076a-b177-41f9-924d-a27680864417` (menu + FAQ)  
**Inbound workflow:** `trigger_call_inbound` ("Inbound restaurant host (live menu + order submit + pay link)", 39 steps). It replaced `trigger_call_inbound_2`. Armed but inert while the agent is a draft. The only open item is that the SMS channel +1 555-555-0123 has no SMS plan (A2P 10DLC not registered).
**Payment-status workflow:** `trigger_webhook` ("Payment status updates from ordering system"). Webhook trigger → in-app notification to DeAli Dillard. Its URL is only generated on activation.
**Ordering-system endpoints:** dummy only, `https://pos-adapter.example.invalid/v1` with token `DUMMY_POS_TOKEN_NOT_REAL`. Nothing can be sent.  
**Transfer target:** +1 555-555-0124  

## What it does
- Answers inbound calls as **Mira** for fictional **Willow & Wick Kitchen** (814 Market Street, Millbrook, SD).
- Menu Q&A, pickup/delivery orders with modifiers, upsell (one max), allergy-safe answers, reservations, human transfer, end-of-flow call summary generation.
- **Live menu check (vendor neutral):** loads the restaurant's live menu at the start of the call, checks each item, sold-out status, price, and allowed options, and offers alternatives. Falls back to the knowledge-base menu with "staff will confirm prices".
- **Cart validation, then a read-back with formula-computed totals.** The tax-math bug is fixed: burger + cheddar now reads $16.50 / $1.07 / $17.57.
- **Order submission** to the restaurant's system after a clear yes. If it fails, staff get a notification with the ticket and the caller hears "staff will confirm shortly". The agent never claims the order is placed unless it was accepted.
- **Compliant payment:** never takes cards by voice. After acceptance it offers a secure text link (with consent) or pay at pickup or the door, and stops callers who read card numbers. Payment status comes back by webhook.

## Files in this folder
| File | Contents |
|------|----------|
| RESEARCH.md | Best-practice research (Vapi/Retell/Slang/Loman + Votel) |
| MENU.md | Full fictional menu (~20+ items), hours, allergens, delivery, reservations |
| FAQ.md | Short policies |
| SYSTEM_PROMPT.md | Polished system prompt (also live on the agent) |
| OPENING_GREETING.txt | Voice greeting (spoken by workflow `greet` step) |
| inbound_workflow_plan.json | Workflow plan used on Votel |
| DEMO_CALL_PATH.md | Why prior calls failed + cheapest path to a recorded demo |
| POS_INTEGRATION.md | Ordering-system adapter spec: menu, cart, order, payment-link, and webhook schemas, vendor notes, compliance |
| TESTS.md | Dry-run results + fixes (rounds 1–3) |
| build/ | Plan generators (gen_plan.py, gen_harness.py), backups of v2 files, final workflow snapshot |
| tests/*.txt | Raw dry-run outputs |

## Untouched agents (Ali / existing)
Inbox Triage, Outbound Campaign Agent (Template), Pest Control Receptionist: not modified. No HVAC agent was in the account as of 2026-10-08 12:16 PM CT.

## Next step for Ali
Approve **one** of:
1. `make_ai_phone_call` to +1 555-555-0124 **without** `agent` (fastest recording; ~$0.12–0.20), or  
2. Activate this restaurant agent + route +1 555-555-0123, then Ali calls in.
