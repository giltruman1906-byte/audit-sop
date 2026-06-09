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

---

## Session 3 — 3 June 2026

### DNS Investigation + Resolution (in progress)

**Root cause found:**
- Resend domain `suki-systems.com` showing "Failed" — all 3 DNS records missing from live DNS
- Records were added to Squarespace Custom DNS but Squarespace is NOT authoritative
- Squarespace nameservers are set to Netlify DNS (`dns1-4.p03.nsone.net`) — DNS is controlled by Netlify
- `suki-systems.com` website is live on Netlify (confirmed via `curl` — `server: Netlify`)
- Netlify DNS zone for `suki-systems.com` is orphaned — exists in Netlify infrastructure but not linked to any active team
- Error when adding domain in Netlify: "managed by Netlify DNS on another team"
- Only one Netlify account exists (`yali@suki-systems.com`, team `yali-xizt504`)

**Actions taken:**
- Added TXT record `verified-for-netlify` → `1046177` to Squarespace (for ownership proof)
- Submitted Netlify support ticket **#1046177** requesting DNS zone release to team `yali-xizt504`
- Waiting for Netlify support to release the orphaned zone

**Once Netlify releases the zone:**
1. Add `suki-systems.com` to Netlify team `yali-xizt504`
2. Add the 3 Resend DNS records in Netlify DNS:
   - TXT `resend._domainkey` → `p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC/7Z/bgx2KRThyAiSOHqQoItwoJhT9kpGVVb4O5kUso/aOGG3B4jXBbllSuQdjPMXBRhMbxqUry7gar3nmpb1fCLBAZq+dfullMZQPEkXwwXDXHCG+MoJt3i1ALmI2tWvPcIkxMotcRtp9pnps+ib9+o/FBP9ckBrlIjSIjlLEwQIDAQAB`
   - MX `send` → `feedback-smtp.us-east-1.amazonses.com` (priority 10)
   - TXT `send` → `v=spf1 include:amazonses.com ~all`
3. Hit Restart in Resend → should verify within minutes
4. Run full end-to-end email test

### UI — Suki Brand Fonts Fixed

**Problem:** Archivo font was referenced in all components but never loaded. All text was falling back to system sans-serif.

**Fix (commit `43deed5`):**
- `layout.tsx` — replaced Geist with Archivo + Archivo Black (loaded via `next/font/google`)
- `globals.css` — Suki brand colors as CSS variables (`#EBE7DD` bg, `#1A2332` dark, `#E85A2C` orange, `#F5F2EC` card), removed dark mode override, removed white background
- All 17 component files — replaced hardcoded `'Archivo, sans-serif'` / `'Archivo Black, sans-serif'` strings with `var(--font-archivo)` / `var(--font-archivo-black)` CSS variables
- Page title updated to "Suki Systems — Project Intake"
- Deployed to Vercel on push to main

---

---

## Session 4 — 9 June 2026

### DNS — Fully Resolved
- Netlify account deleted — DNS reverted to Squarespace (now authoritative)
- DKIM record re-added to Squarespace Custom DNS (previous value had embedded spaces — fixed)
- All 3 Resend records now **verified**: SPF MX ✅, SPF TXT ✅, DKIM ✅
- Resend domain `suki-systems.com` status: **Verified**

### Client Approval/Review Step — Built
The flow now has a review gate before emails fire:

**New status flow:** `active → budget → contact → review → complete`

- `app/api/interview/[id]/lead/route.ts` — contact submit now sets status `review` (was `complete`)
- `app/interview/[id]/ReviewStep.tsx` — new component: generates summary preview, shows "Does this look right?", Approve & Submit / Go back buttons
- `app/api/interview/[id]/approve/route.ts` — new route: sends both emails, sets status `complete`, parses provisioning items
- `app/api/interview/[id]/reopen/route.ts` — new route: clears brief, sets status back to `active`
- `app/api/interview/[id]/finalize/route.ts` — now generates + stores only (no emails)
- `app/interview/[id]/page.tsx` — routes `review` status to ReviewStep, `complete` to SummaryView

### Email Bugs Fixed
1. **Fire-and-forget broken on Vercel** — Vercel terminates the process when the response returns, killing unresolved Promises. Fixed by awaiting both emails via `Promise.allSettled` before returning.
2. **Brief never saved** — `supabase.upsert` with `onConflict: 'interview_id'` silently failed because no UNIQUE constraint exists on that column. Fixed with explicit check-then-insert-or-update pattern.

### UI Brand Update
- All navbars flipped: dark navy background → light cream (`#EBE7DD`), matching suki-systems.com
- Subtle dot-grid background texture added to `globals.css`
- Applied across: Chat, Budget, Contact, Review, Summary, and terminated screens

### Vercel Deployment Protection — Disabled
Was blocking clients from reaching the app (redirected to Vercel login). Turned off in Vercel project settings.

### End-to-End Test Run
- Ran full flow as client (Moshe Moshe, `gil@suki-systems.com`)
- Interview → Budget → Contact → Review screen → Approve → SummaryView ✅
- Emails did NOT arrive — root cause: `RESEND_API_KEY` in Vercel is stale/wrong
- Confirmed local key works: test email sent via curl → arrived in `giltruman1906@gmail.com` ✅
- `gil@suki-systems.com` delivery unclear (may not be a real receiving mailbox)

---

---

## Session 5 — 9 June 2026 (continued)

### Email delivery fully resolved ✅
- `RESEND_API_KEY` updated in Vercel — confirmed working
- Root cause of `yali@suki-systems.com` / `gil@suki-systems.com` not receiving: missing MX records
- DNS is controlled by **Vercel DNS** (not Squarespace) — MX records must go in Vercel
- Added 5 Google Workspace MX records to Vercel DNS:
  - Priority 1: `ASPMX.L.GOOGLE.COM`
  - Priority 5: `ALT1/ALT2.ASPMX.L.GOOGLE.COM`
  - Priority 10: `ALT3/ALT4.ASPMX.L.GOOGLE.COM`
- Both addresses confirmed delivering ✅
- End-to-end test passed: full flow → both emails arrived

### Flow + UI polish
- Claude closing message fixed: no bullet-point recap, warm 2-sentence handoff
- SummaryView: "Phase 1 Complete — We're on it." with pulsing dots
- ContactPopup: step 3 of 4, button "Continue to review →"
- BudgetStep: step 2 of 4
- Dashboard agency brief: markdown rendered properly (was raw `<pre>`)

---

## Next Session Priorities

1. M9 — acceptance tests from §11 of BUILD_BRIEF_Final.md
2. Design review — further alignment with suki-systems.com brand
3. Custom domain on Vercel (optional)

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
