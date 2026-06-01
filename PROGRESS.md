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

### What's next — M1 (Interview Loop)

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
