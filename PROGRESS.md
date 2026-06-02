# Intake Engine — Build Progress

## Session 1 — 1 June 2026

### What we built

#### Repos created
- **Private:** `giltruman1906-byte/intake-engine` — the app
- **Public:** `giltruman1906-byte/audit-sop` — SOPs and build docs

#### M0 — Scaffold (COMPLETE)

**Next.js app**
- Next.js 15, App Router, TypeScript, Tailwind
- Folder structure matches §4 of the build brief
- Root redirects to `/dashboard`

**Supabase**
- `lib/supabase.ts` — browser client, server client, admin client (service role, server-only)
- Auth middleware (`middleware.ts`) — protects `/dashboard`, allows `/login` and `/interview`
- `db/schema.sql` — all 8 tables created + RLS policies applied + Suki seeded ✓

**Database tables (all live in Supabase)**
- `agencies` — Suki Systems seeded as agency #1 (`id: 00000000-0000-0000-0000-000000000001`)
- `agency_credentials` — encrypted BYO API keys per tenant
- `pricing_tiers` — Suki seeded with Small / Medium / Large tiers
- `interviews` — one row per intake session
- `messages` — transcript (role + content)
- `leads` — contact info captured at submit
- `briefs` — agency markdown + client summary markdown
- `provisioning_items` — accounts + env var names + vault locations

**Pages built**
- `/login` — Suki-branded auth page
- `/dashboard` — lists interviews, links to rate card + new interview
- `/dashboard/rate-card` — shows pricing tiers

**Email**
- `lib/email/client-summary.html` — branded HTML email template (Suki design system)
- `lib/email/render-template.ts` — fills `{{placeholders}}` at send time

**Environment**
- `.env.local` — all keys filled in (Supabase, Anthropic, Resend)
- `.env.example` — safe template committed to git
- `ENCRYPTION_KEY` and `SESSION_SECRET` still need to be generated (run: `node -e "require('crypto').randomBytes(32).toString('hex')"` twice)

---

---

## Session 2 — 2 June 2026

### What we built — M1 (Interview Loop) COMPLETE

**New files**
- `lib/interview-spec.ts` — system prompt + 15-field schema + `isInterviewComplete()` (core IP)
- `lib/claude.ts` — provider-abstracted Anthropic client; swap API key here for BYO key (M7)
- `app/api/interview/start/route.ts` — GET → creates interview row → redirects to `/interview/{id}`
- `app/api/interview/[id]/message/route.ts` — POST → streams Claude response as SSE; persists transcript; detects completion via `save_fields` tool call; advances status to `budget` when done
- `app/interview/[id]/page.tsx` — server component; loads interview + messages; renders ChatClient
- `app/interview/[id]/ChatClient.tsx` — client component; handles SSE streaming, typing indicator, auto-triggers opening message on first load, shows completion banner

**Refactor**
- Split `lib/supabase.ts` (browser-only) from `lib/supabase-server.ts` (server + admin) — was causing a build error because `next/headers` was being pulled into the client bundle via the login page

**How completion detection works**
- System prompt instructs Claude to call `save_fields` tool after every response, with all 15 fields (null if not yet captured)
- Message route accumulates `input_json_delta` events from the stream
- After stream ends, parses tool JSON → `isInterviewComplete()` → if all 15 filled, sets `status = 'budget'`
- Client receives `{t:"done", complete:true/false}` as final SSE event

**Tested manually**
- `/api/interview/start` → creates row in Supabase, redirects to chat page ✓
- Opening message streams correctly ✓
- Follow-up user message → Claude asks ONE next question ✓
- `{t:"done", complete:false}` received correctly ✓

---

### What we built — M2 (Budget Step) COMPLETE

**New files**
- `lib/pricing.ts` — `getMatchedTier(complexitySignals, tiers)`: runs deterministic keyword rules first (enterprise/multi-tenant → large; single workflow → small); falls back to Claude `classify_tier` tool for ambiguous cases. Never invents prices — always maps to agency rate card.
- `app/api/interview/[id]/budget/route.ts` — POST `{tier}`: verifies tier exists on the agency's rate card; saves `tier`, `budget_min`, `budget_max`, `estimated_timeframe` to `interviews`; advances status to `contact`
- `app/interview/[id]/BudgetStep.tsx` — client component; shows all three tier cards with suggested tier pre-selected; disclaimer "indicative estimate, not a binding quote"; sign-off button posts to budget route; on success calls `router.refresh()` to advance

**Modified files**
- `app/api/interview/[id]/message/route.ts` — on completion: fetches agency's tiers, runs `getMatchedTier`, saves suggested tier to interview; pricing failure is non-fatal (still advances to budget)
- `app/interview/[id]/page.tsx` — now routes by status: `active` → ChatClient, `budget` → BudgetStep, `contact` → placeholder (M3), `complete` → done screen
- `app/interview/[id]/ChatClient.tsx` — completion banner now has "See budget range →" button that calls `router.refresh()` to re-render server component

**How it all fits**
1. Interview completes → message route runs pricing → saves suggested tier → sets status='budget'
2. ChatClient shows completion banner with button
3. User clicks → router.refresh() → page.tsx re-fetches → renders BudgetStep with pre-selected tier
4. User adjusts if needed → clicks sign-off → budget route validates against rate card → saves → status='contact'
5. router.refresh() → page.tsx shows contact placeholder (M3 next)

---

### What we built — M3 (Contact Capture) COMPLETE

**New files**
- `app/interview/[id]/ContactPopup.tsx` — full-page contact form; purpose statement shown above the fields ("Suki Systems will use these details to follow up…"); company name, contact name, email (required), phone (optional); validates email format client- and server-side; `consent_ts` set server-side on submit
- `app/api/interview/[id]/lead/route.ts` — validates status='contact'; saves to `leads` with `consent_ts = now()`; sets interview status='complete' and `completed_at`

**Modified files**
- `app/interview/[id]/page.tsx` — contact placeholder replaced with `<ContactPopup />`

**Guardrails verified**
- Collects exactly: company_name, contact_name, email, phone — nothing more
- Purpose statement visible before any data entry
- consent_ts set server-side (not client-supplied)
- Phone is optional; all other fields required

---

### What's next — M4 (Finalize → Dual Output)

Build order:
1. `lib/brief-generator.ts` — transcript + leads → Claude-Code-ready agency brief (Markdown, §7a structure)
2. `lib/summary-generator.ts` — transcript + budget → plain-language client summary (§7b structure)
3. `app/api/interview/[id]/finalize/route.ts` — POST: generates both outputs, stores in `briefs` table
4. `app/interview/[id]/SummaryView.tsx` — shows client summary on screen after submit

Done-criteria for M4:
- Both outputs match §7a/§7b structure exactly
- Neither contains any secret value (guardrail #1)
- Summary labels timeframe + budget as INDICATIVE
- Both stored in `briefs` table before being displayed/sent

---

### What was next before (M1 plan, now done) — kept for reference

### What's next — M1 (Interview Loop) [DONE]

The heart of the product. Build order:

1. `lib/interview-spec.ts` — system prompt + schema (the core IP)
2. `app/api/interview/start/route.ts` — creates a new interview row, returns `id`
3. `app/api/interview/[id]/message/route.ts` — streams Claude follow-up questions
4. `app/interview/[id]/page.tsx` — chat UI (text input, streaming responses)
5. Transcript persists to `messages` table after each turn
6. Engine signals when all required fields are filled

**Required fields the interview must extract:**
`project_name · what_they_want · current_process_steps · data_sources · where_data_lives · trigger_and_frequency · inputs · outputs · who_uses_output · integrations_needed · constraints · success_criteria · edge_cases · rough_volume · complexity_signals`

**Done-criteria for M1:**
- Interview asks ONE leading question at a time
- Follows up on vague answers
- Stops only when every required field is filled
- Transcript persisted to `messages` table

---

### Guardrails (non-negotiable — check on every change)
1. No secret ever in a brief, summary, email, transcript, or log
2. Never auto-create accounts for a client
3. API keys encrypted at rest, decrypted server-side only, never logged
4. `SUPABASE_SERVICE_ROLE_KEY` server-only — never in the client bundle
5. Budget = rate-card driven, never LLM-invented
6. Contact PII stored securely, purpose statement shown to client
7. Stop and ask before anything touching auth, billing, key handling, PII, or deletion

---

### Key file locations
| File | Purpose |
|---|---|
| `intake-engine/db/schema.sql` | Run this in Supabase SQL Editor to (re)create tables |
| `intake-engine/lib/supabase.ts` | Supabase clients |
| `intake-engine/lib/interview-spec.ts` | **TODO M1** — core IP |
| `intake-engine/lib/pricing.ts` | **TODO M2** — scope→tier mapping |
| `intake-engine/middleware.ts` | Auth protection |
| `intake-engine/.env.local` | Local secrets (never committed) |
| `intake-engine/lib/email/client-summary.html` | Branded email template |
