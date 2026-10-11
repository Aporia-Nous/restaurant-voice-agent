# Restaurant Voice-Agent Research Notes
Date: 2026-10-08 (America/Chicago)

## Sources reviewed
- DeskCaller: "How we write system prompts for restaurant agents" (Aug/Sep 2026)
- Stable Kernel: Menu modifiers & edge cases in voice ordering
- Tirnav: Vapi/Retell restaurant + POS integration guide
- Retell AI prompting principles
- Google Cloud Food Ordering AI Agent menu model
- Votel docs / MCP tool schemas; AllThingsVotel first-agent + MCP guides
- Votel pricing (voice ~$0.039/min platform + STT/TTS; example all-in ~$0.058/min)

## Cross-platform best practices (Vapi, Retell, Slang, Loman patterns)

### Conversation style
- Warm host persona; short spoken sentences (1–2 per turn); one question at a time
- No markdown, bullets, emoji — this is speech
- Spoken-form prices/times/numbers ("twelve ninety-five", "twenty-five minutes")
- Confirm party size + time together to catch two/four, eight/ate homophones
- Stop immediately on interruption; never resume the cut-off sentence
- Match caller pace (rushed → brief; unsure → slower)

### Order taking
- Capture items one at a time with exact modifiers
- Confirm each item briefly as added ("Got it — one Margherita, no basil")
- Ask for required modifiers before moving on (size, protein, spice)
- One relevant upsell max (side or drink), never pressure
- Final full read-back: quantities, modifiers, fulfillment, total + tax estimate, time
- Explicit yes before confirming the order

### Allergies & dietary
- State listed allergens only; NEVER say "safe", "allergen-free", or "free from"
- Disclose shared kitchen / cross-contact limits
- Always: "Please tell our team about your allergy when you arrive"
- Severe allergy or uncertainty → transfer to a human with context

### Payment
- Never collect card numbers, CVV, or full payment data by voice
- Pay at pickup or on delivery (cash/card with driver/counter)

### Busy / edge cases
- Mid-order corrections: update the cart, re-confirm changed item
- 86 / unavailable: apologize, offer alternatives, never invent
- Noise / mishear: closed clarification ("four or five?"), two tries then transfer
- Complaints, catering, private events, refunds → transfer

### Reservations
- Collect date, time, party size, name, phone one at a time
- Large parties: note deposit/set-menu policy, may need human confirm
- Never invent availability; if no live booking tool, take a request and confirm staff will verify — or transfer

## Votel-specific
- Agents are draft until `activate_ai_agent`; numbers won't route to drafts
- `make_ai_phone_call` can omit `agent` to build a temporary single-use agent from `goal` (bypasses activation catch-22 for outbound demos)
- Recordings: `get_call_recording_link`; transcripts: `get_call_transcript`; status: `check_ai_phone_call` / `get_call`
- Inbound: route number to agent (Settings → Phone or `update_phone_number` destination_type agent) AFTER activate
- Knowledge: `configure_agent_knowledge` text snippets / files; keep facts reviewable separately from behavior
- Transfer: workflow `call_forwarding` (external number) or global talking node → forward
- Voice: Cartesia / ElevenLabs; Cartesia Skylar used elsewhere on account; Katie Cartesia is a strong warm host option
- Dry-run: `test_agent_workflow` is non-billing (side effects recorded, not performed)
