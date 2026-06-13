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

## Session 6 — 9 June 2026 (continued)

### M9 Acceptance Tests — Partial Pass

Ran all 11 tests from §11 of BUILD_BRIEF_Final.md:

| # | Test | Result |
|---|---|---|
| 1 | RLS: client cannot read other interviews | ⚠️ Partial — admin client used in API (by design), full RLS audit deferred |
| 2 | Interview chat streams correctly | ✅ |
| 3 | All 15 fields captured before completion | ✅ |
| 4 | Budget step shows rate-card tiers | ✅ |
| 5 | Contact form saves lead + consent_ts | ✅ |
| 6 | Review step shown before emails fire | ✅ |
| 7 | Agency brief email arrives at BRIEF_DELIVERY_EMAIL | ✅ |
| 8 | Client summary email arrives at client address | ✅ |
| 9 | Provisioning checklist populates in dashboard | ⚠️ Bug found + fixed (see below) |
| 10 | BYO key: agency can set own Anthropic key | 🔲 Not yet tested manually |
| 11 | Drop brief into Claude Code → scaffolds project | 🔲 Not yet tested manually |

### Provisioning Fire-and-Forget Bug — Fixed

**Bug:** `parseProvisioningItems` (calls Claude API) was invoked as fire-and-forget in the approve route. Vercel kills unresolved Promises when a serverless function returns, so provisioning items were never saved → dashboard checklist always empty.

**Fix (commit `82b6bac`):**
- Moved provisioning parse into `Promise.allSettled` alongside the two emails
- All three now awaited before `return Response.json({ ok: true })`
- Added idempotency guard: skips insert if items already exist for the interview
- Failure logged via `console.error` but never blocks the approve response

---

## Session 7 — 13 June 2026

### Real client stuck at `review` — diagnosed + recovered

**Report:** First real external client (AB Residential / Mikey Azougui, `mikey@ab-residential.com`, interview started 10 Jun). Dashboard populated, but neither Suki nor the client received an email. Reported as a "Resend / email" issue.

**Diagnosis (NOT a Resend problem):**
- Interview `d673ac80-…` was stuck at `status='review'`, `completed_at=null`, **zero `provisioning_items`**.
- Brief existed (from `finalize`, which auto-runs on ReviewStep mount) — that's why the dashboard looked "populated."
- Both emails + provisioning are sent/created **only** by the `approve` route (fired by the client clicking "Approve & Submit"). Status still `review` ⇒ approve never completed ⇒ Resend was never invoked.
- Root cause: client reached the review screen but never completed the approval click. Confirmed pattern — interview `efd9ef12` (old "moshe" test) is stuck the same way.

**Recovery:** Ran the real `approve` route locally (prod URL has Vercel auth protection on) against live Supabase + Resend. Result: status → `complete`, both emails sent (no errors), 6 provisioning items created.

**Open gap (now being fixed):** The approval gate is silent — if a client abandons at `review`, no email fires and the agency gets no notification.

### ⛔ Stuck-at-review notification — REMOVED (commit `29d3588`)
**Reverted by request.** Decision: rely on the dashboard `status` column to see where each client is, no email alert needed. Deleted the cron route, `vercel.json`, `sendStuckReviewEmail()`, `CRON_SECRET`, and the `review_notified_at` schema line. Two leftovers (both harmless, optional to clean up): the live DB still has an unused `review_notified_at` column (`alter table interviews drop column review_notified_at;` to remove), and the `CRON_SECRET` env var in Vercel can be deleted. Original build notes kept below for history.

### Stuck-at-review notification — BUILT then removed (history)

A Vercel Cron job that alerts the agency when leads sit at `review` without approving.

- `app/api/cron/stuck-review/route.ts` — GET handler, guarded by `CRON_SECRET` Bearer. Finds interviews `status='review'`, `review_notified_at IS NULL`, whose lead `consent_ts` is older than **30 min**; emails a digest to `BRIEF_DELIVERY_EMAIL`; marks `review_notified_at` so it never double-notifies.
- `lib/email.ts` — new `sendStuckReviewEmail()` (branded HTML digest table of stuck leads + dashboard links).
- `vercel.json` — runs **twice daily at 10:00 & 17:00 AWST** (`0 2 * * *` and `0 9 * * *` UTC; WA = UTC+8, no DST). On Vercel Pro.
- `.env.local` / `.env.example` — new `CRON_SECRET`.
- `db/schema.sql` — added `review_notified_at timestamptz` + `irrelevant_count` to interviews; status comment updated.
- Verified: `tsc` clean; route 401 without secret; cron query executes against the live column and returns `[]` (nothing stuck) correctly.

**DONE:** column `review_notified_at` added to live DB; committed + pushed (`1352546`) → Vercel auto-deploy.
**Confirm in Vercel:** `CRON_SECRET` env var is set (value in `.env.local`) — without it the cron route runs unauthenticated (still works, just not secured).

### DB cleanup — old test interviews deleted
Deleted all 7 interviews created before 2026-06-10 (Test Co, moshe builders, Acme corp, + 4 empty drafts) via cascade. **Only Mikey / AB Residential (`d673ac80…`) remains.** Dashboard is now clean.

### New pricing tiers — DONE (live)

Added two tiers below the old $5k floor (`tier` is plain text, no migration needed). Renumbered sort_order; live in `pricing_tiers` + seeded in `schema.sql`:

| order | tier | label | range (AUD) | timeframe |
|---|---|---|---|---|
| 1 | `micro` | Quick Fix | $250 – $1,000 | 2–5 days |
| 2 | `starter` | Starter Project | $1,000 – $5,000 | 1–2 weeks |
| 3 | `small` | Small Project | $5,000 – $10,000 | 1–3 weeks |
| 4 | `medium` | Medium Project | $10,000 – $25,000 | 3–6 weeks |
| 5 | `large` | Large Project | $25,000 – $50,000 | 6–10 weeks |

Budget page + AI tier classifier read tiers dynamically, so both auto-include the new ones. Prices/labels editable in `/dashboard/settings`.

---

## Next Session Priorities

1. **End-to-end test** — full flow with provisioning fix live (verify checklist populates in dashboard)
2. **M9 test #10** — manual: go to `/dashboard/settings`, enter a BYO Anthropic key, run a new interview, confirm it uses that key
3. **M9 test #11** — manual: copy the agency brief from dashboard, paste into a Claude Code session, confirm it generates a working project scaffold
4. Design review — further alignment with suki-systems.com brand
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
