# Intake Engine — Build Progress

## Session 1 — 1 June 2026

### What we built — M0 (Scaffold) COMPLETE

#### Repos created
- **Private:** `giltruman1906-byte/intake-engine` — the app
- **Public:** `giltruman1906-byte/audit-sop` — SOPs and build docs

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
- `.env.local` — all keys filled in (Supabase, Anthropic, Resend, ENCRYPTION_KEY, SESSION_SECRET)
- `.env.example` — safe template committed to git

---

## Session 2 — 2 June 2026

### M1 — Interview Loop COMPLETE

- `lib/interview-spec.ts` — system prompt + 15-field schema + `isInterviewComplete()` (core IP)
- `lib/claude.ts` — provider-abstracted Anthropic client
- `/api/interview/start` → POST, creates interview row, returns `{id}`
- `/api/interview/[id]/message` → streams Claude response as SSE; persists transcript; detects completion via `save_fields` tool; advances status to `budget` when done
- `/interview/[id]` → server-rendered page + ChatClient streaming chat UI
- Split `lib/supabase.ts` (browser) from `lib/supabase-server.ts` (server+admin)

### M2 — Budget Step COMPLETE

- `lib/pricing.ts` — deterministic keyword rules + Claude `classify_tier` tool for ambiguous cases; always maps to agency rate card
- `/api/interview/[id]/budget` — validates tier against rate card, saves sign-off, advances to `contact`
- `BudgetStep.tsx` — tier cards with suggested pre-selected, indicative disclaimer, sign-off button
- Pricing now runs fire-and-forget after setting status='budget' (timing fix)

### M3 — Contact Capture COMPLETE

- `ContactPopup.tsx` — company name, contact name, email (pre-filled from auth), phone; purpose statement shown before submit
- `/api/interview/[id]/lead` — saves to `leads` with `consent_ts`, advances to `complete`

### M4 — Finalize → Dual Output COMPLETE

- `lib/brief-generator.ts` — §7a agency brief; no secrets, env var names + vault locations only
- `lib/summary-generator.ts` — §7b client summary; indicative disclaimer enforced
- `/api/interview/[id]/finalize` — idempotent; generates both in parallel; upserts to `briefs`
- `SummaryView.tsx` — triggers finalize on mount; shows branded confirmation message

### M5 — Email Delivery COMPLETE

- `lib/email.ts` — Resend wrapper; `sendBriefEmail` (agency internal), `sendSummaryEmail` (client branded)
- Client email now uses the Suki-branded HTML template from `lib/email/client-summary.html`
- Template sections parsed from summary Markdown and injected with structured data
- Fire-and-forget after storing brief — email failure never blocks the client

### M6 — Provisioning Checklist COMPLETE

- `lib/provisioning-parser.ts` — Claude tool-use extracts service/env_var_name/vault_location
- `/api/interview/[id]/provision` — PATCH toggles item pending/done
- `/dashboard/interview/[id]` — detail page: meta, lead, checklist with checkboxes, full brief
- `ProvisioningChecklist.tsx` — interactive client component, optimistic toggle

### M7 — BYO Key + Rate-Card Config UI COMPLETE

- `lib/crypto.ts` — AES-256-GCM encrypt/decrypt for tenant API keys
- `lib/claude.ts` — `getAgencyApiKey()` decrypts stored key, falls back to env var
- `/api/agency/key` — GET/POST/DELETE BYO key (never returns the key to client)
- `/api/agency/rate-card` — PUT to update a pricing tier
- `/dashboard/settings` — ApiKeyPanel + RateCardEditor UI

### Post-M7 additions (same session)

**Vercel deploy**
- Live at `intake-engine-881jqdx0l-suki-systems.vercel.app`
- All 9 env vars set in Vercel dashboard
- Agency login: `giltruman1906@gmail.com`

**Client auth**
- `/interview/[id]/auth` — client sign-up/login page (email + password)
- Middleware updated: `/interview/{id}` requires auth, redirects to interview-specific auth page
- ContactPopup pre-fills email from auth session (read-only)

**Bug fixes**
- 404 for clients: switched admin Supabase client to raw `supabase-js` (was failing without cookies)
- Dashboard empty: switched to admin client + SUKI_AGENCY_ID scope (RLS JWT claim not configured)
- Flow not advancing to budget: status set immediately, pricing runs in background

**Session management**
- `FLAG_IRRELEVANT_TOOL` added to interview spec
- `irrelevant_count` column added to interviews table
- After 5 irrelevant flags → status='terminated'; SSE sends `{t:"terminated"}`
- Termination overlay popup in ChatClient with "Contact Suki Systems" mailto link
- Input locked permanently after complete or terminated

**Upsell persona**
- System prompt updated: Claude spots adjacent automation opportunities and mentions once naturally

**Post-submit UX**
- SummaryView loading state shows: "We have everything we need. Report coming to your email..."

**DNS (pending)**
- 3 Resend DNS records added to Squarespace Custom Records
- Waiting for NS1 propagation (up to 24h) — records show "Pending" in Resend

---

## Next Session Priorities

1. Confirm Resend DNS verified (check resend.com → Domains → suki-systems.com)
2. Run full end-to-end test with real emails
3. M9 — acceptance tests from §11 of BUILD_BRIEF_Final.md
4. Fix any issues found in testing
5. Custom domain on Vercel (optional)

---

## Key File Locations

| File | Purpose |
|---|---|
| `intake-engine/db/schema.sql` | Full Postgres schema |
| `intake-engine/lib/interview-spec.ts` | Core IP — interview prompt + field schema |
| `intake-engine/lib/pricing.ts` | Scope→tier mapping |
| `intake-engine/lib/crypto.ts` | AES-256-GCM key encryption |
| `intake-engine/lib/email/client-summary.html` | Suki-branded client email template |
| `intake-engine/lib/email/render-template.ts` | Template renderer |
| `intake-engine/middleware.ts` | Auth protection |
| `intake-engine/.env.local` | Local secrets (never committed) |
| `DEPLOY_STATUS.md` | Live deployment status + DNS records |
