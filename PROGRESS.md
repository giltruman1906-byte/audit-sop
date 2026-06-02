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

### What we built — M4 (Finalize → Dual Output) COMPLETE

**New files**
- `lib/brief-generator.ts` — generates full technical agency brief (§7a); system prompt enforces: no secret values, only env var names + vault locations; budget labeled indicative
- `lib/summary-generator.ts` — generates plain-language client summary (§7b); must label timeframe + budget as "indicative estimate — not a binding quote"
- `app/api/interview/[id]/finalize/route.ts` — POST; verifies status='complete'; returns cached brief if already generated (idempotent); gathers interview + transcript + lead + agency + tier; generates both outputs in parallel (Promise.all); upserts to `briefs` table
- `app/interview/[id]/SummaryView.tsx` — client component; triggers finalize on mount if no cached brief; shows loading dots; renders summary as styled sections; caches and skips on refresh

**Modified files**
- `app/interview/[id]/page.tsx` — 'complete' status now fetches cached brief and renders SummaryView (or triggers generation if none)

**Tested manually** — finalize called against real interview data:
- Client summary: plain language, correct structure, indicative disclaimer present ✓
- Agency brief: technical, structured, includes requirements/stack/architecture/build plan ✓
- Both generated in parallel, stored in `briefs` table, cached on second call ✓

---

### What we built — M5 (Email Delivery) COMPLETE

**New files**
- `lib/email.ts` — Resend wrapper; `sendBriefEmail()` sends agency brief to `BRIEF_DELIVERY_EMAIL`; `sendSummaryEmail()` sends client summary to lead email; `extractProjectName()` pulls the # heading from the brief for the subject line; both convert Markdown to a styled HTML email

**Modified files**
- `app/api/interview/[id]/finalize/route.ts` — after storing the brief, fires both emails as fire-and-forget (`.catch` logged, never blocks); email failures don't prevent the summary from displaying to the client

**Guardrails verified**
- Brief email: contains Markdown doc content only — no API keys, no secrets, no session tokens
- Neither email is sent from the block that generates content — they're dispatched after saving to DB
- `BRIEF_DELIVERY_EMAIL` is read server-side only from env, never exposed to client

---

### Milestones M0–M5 complete. Core beta flow is functional end-to-end.

The full client journey now works:
1. Agency starts interview → shares link
2. Client answers questions (M1)
3. Budget range shown, client signs off (M2)
4. Contact details captured (M3)
5. Brief + summary generated (M4)
6. Brief emailed to agency, summary emailed to client (M5)

---

### What we built — M6 (Provisioning Checklist) COMPLETE

- `lib/provisioning-parser.ts`: Claude tool-use extracts service/env_var_name/vault_location from brief
- finalize route: saves items to `provisioning_items` after generation (fire-and-forget)
- `/api/interview/[id]/provision`: PATCH toggles item status pending/done
- `/dashboard/interview/[id]`: interview detail page — meta, lead, checklist, brief text
- `ProvisioningChecklist.tsx`: interactive checkboxes, optimistic toggle
- Dashboard rows link to detail page

### What we built — M7 (BYO Key + Rate-Card Config UI) COMPLETE

- `lib/crypto.ts`: AES-256-GCM encrypt/decrypt; ENCRYPTION_KEY from env (throws if missing)
- `lib/claude.ts`: `getAgencyApiKey()` — decrypts stored key, falls back gracefully
- Brief + summary generators: accept optional `apiKey` prop
- Message + finalize routes: fetch BYO key per agency before Claude calls
- `/api/agency/key`: GET (configured?), POST (encrypt+store), DELETE — key never returned to client
- `/api/agency/rate-card`: PUT to update a pricing tier (scoped to agency)
- `/dashboard/settings`: ApiKeyPanel + RateCardEditor — full config UI

---

### M0–M7 complete. All core features built.

**What remains before Vercel deploy:**
- M8 — Voice input (optional, skip for now)
- M9 — Polish + acceptance tests (§11)
- Vercel deploy + env var setup
- Generate ENCRYPTION_KEY and SESSION_SECRET in `.env.local`

### What's next — M6 (Provisioning + Vault Handoff) [DONE — see above]

### What was next — M6 (Provisioning + Vault Handoff)

Build order:
1. After finalize, parse the provisioning checklist from the agency brief and save rows to `provisioning_items`
2. Show the checklist on the dashboard (for the agency to track which accounts the client has created)

Done-criteria for M6:
- Each provisioning item has: service, env_var_name, vault_location, status=pending
- No actual secret values in provisioning_items
- Dashboard shows checklist; agency can mark items done

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
