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

## Session 8 — 13 June 2026

### 2nd real client (Invictus Institute) — stuck at `active`, recovered manually

**Report:** Client Benjamin (`benjamin@invictushealthhub.com`, company "Invictus Institute") completed the full intake conversation for interview `b60b94bf-…` but no email arrived and the flow never advanced. Different failure from Session 7's stuck-at-`review`.

**Diagnosis — two separate bugs:**

1. **Completion never fired (root cause).** Interview was stuck at `status='active'`. Transcript was complete (all 15 fields richly captured) and Claude even gave its verbal closing ("That's everything we need… On to the next step!" at msg 34) — but the `save_fields` tool call on that turn did **not** mark all 15 fields non-null, so `isInterviewComplete()` never returned true and status never flipped `active`→`budget`. The client was left on the chat screen with no "next" button and asked "anything else you need to get this sent off?". **The verbal closing and the actual completion signal are decoupled** — the model can *say* it's done without the tool reflecting it. Not yet fixed in code (see Next Priorities).

2. **Brief truncated at `max_tokens: 4096`.** Found during recovery: `lib/brief-generator.ts` capped output at 4096 tokens. This large project (8 programs, ~10 integrations) blew past it — the agency brief was cut off mid-"Build Plan", dropping Provisioning Checklist, Budget, Client Contact, Acceptance Criteria, Open Questions. That's also why `parseProvisioningItems` returned 0 (no checklist section to parse). **Fixed (local, uncommitted):** raised to `max_tokens: 16000` + added a `stop_reason === 'max_tokens'` guard that throws instead of silently storing a truncated brief.

**Manual recovery (ran real route code via local dev server against live Supabase + Resend):**
- Flipped `active`→`budget`, then `budget` route with **Large** tier ($25k–$50k AUD, 6–10 weeks)
- `lead` route: company "Invictus Institute", contact "Benjamin", email `benjamin@invictushealthhub.com` (email recovered from the auth account created at interview start)
- `finalize` → first pass produced truncated brief; after the max_tokens fix, deleted the cached brief and re-ran → **26,206-char complete brief** (all 10 sections)
- `approve` → sent client summary email, marked `complete`, but provisioning still 0 (truncated brief at that point)
- One-off temp route → parsed **22 provisioning items** from the complete brief + **re-sent the complete agency `.md`** to yali@ (client NOT re-emailed — already had correct summary). Temp route deleted after.

**Final state (verified):** status `complete`, tier large, lead saved, agency brief 26.2k chars, client summary 3.9k chars, 22 provisioning items (all pending). Both emails confirmed sent, no Resend errors.

### Robustness pass — completion gate + timeout fixed (same session, tested)

Triggered by the Invictus failure + an imminent 3rd customer ("no issues"). Full flow audit → fixed both the root completion bug and a newly-found timeout risk. All `tsc` clean, `next build` passes, behaviour verified via API tests.

**1. Completion gate — no longer strandable (root fix).** Previously `active`→`budget` depended ENTIRELY on the model emitting a perfect 15-field `save_fields` on the closing turn; if it didn't, the client was stuck with no escape. Now THREE independent triggers, any one advances:
- `lib/interview-spec.ts` — new `COMPLETE_INTERVIEW_TOOL`; prompt updated to require calling it on the closing turn ("call even if unsure every box is filled").
- `app/api/interview/[id]/message/route.ts` — advances on `complete_interview` **OR** full `save_fields`; advance logic moved to shared `advanceToBudget()` and **awaited** (no more fire-and-forget pricing that Vercel could kill).
- `lib/complete-interview.ts` (new) — shared `advanceToBudget()` + `extractFieldsFromTranscript()` (forced server-side re-extraction over the whole transcript, independent of inline tool calls).
- `app/api/interview/[id]/advance/route.ts` (new) — safety-net endpoint: re-extracts from transcript, advances if complete, returns `{reason:'incomplete'|'too_early', missing:[...]}` otherwise. Idempotent (no-op if already past `active`). Guards against advancing a <4-user-turn chat.
- `app/interview/[id]/ChatClient.tsx` — subtle "Already covered everything? Continue →" escape-hatch link, shown only after ≥6 user turns when not yet complete; calls `/advance`.

**2. Vercel timeout risk (newly found, critical).** No `maxDuration` set anywhere + no `vercel.json`. `finalize` ran 82–138s locally (worse now with the 16k brief) → would hit Vercel's default timeout in prod → client stuck forever on the review spinner. Added `export const maxDuration = 300` to `finalize`, `approve`, and `message` routes. **Requires Vercel Pro** (they are, per session 7) — if not on a plan allowing 300s the cap still applies.

**Tested (local dev + live Supabase):** `/advance` on a copy of Benjamin's full transcript → `advanced:true`, tier auto-set to large, status→budget. Barely-started chat → `too_early` 409. Second call → idempotent `advanced:false`. Test interviews cleaned up.

### Shipped to production + live prod smoke test (same session)

All fixes committed and pushed to `main` → Vercel auto-deployed. Commits:
- `aab6731` — completion gate (server-side hand-off detection + `complete_interview` tool + `/advance` route + UI escape hatch) + brief truncation (max_tokens 16k) + `maxDuration=300` on message/finalize/approve.
- `5c09dde` — dashboard Client column now reads the company name from `leads` (was showing `—` from the never-populated `interviews.client_label`).
- `19185b1` — provisioning parser max_tokens 1024→4096 (see below).

**KEY FINDING — completion can't depend on the model.** Live test proved claude-sonnet-4-6 routinely gives a perfect closing message ("…on to the next step!") while firing NEITHER completion tool. Fix: server detects the hand-off (assistant reply has no `?` + ≥6 user turns) → re-extracts fields from the full transcript → advances. Model tool calls are now just a fast-path, not the gate.

**KEY FINDING — "max_tokens too low" is a recurring silent-failure pattern.** THREE spots truncated with no error, each yielding empty/half output: brief-generator (4096→16000), and provisioning-parser (1024→4096 — 20+ env vars overflowed the tool JSON → empty checklist for every large project). Only caught by the live smoke test. All now guard `stop_reason==='max_tokens'`. Rule: any new Claude call emitting structured/long output needs generous max_tokens + a stop_reason guard.

**Live prod smoke test — PASSED end-to-end.** Fresh interview on `intake-engine-umber.vercel.app` through every stage: intake auto-advanced → budget (large) → contact (`liaba99@gmail.com`) → finalize (30k brief, **159s** — under the new 300s cap, would've died on the old default) → approve → `complete`, 28 provisioning items, both emails delivered (user confirmed). The smoke test itself surfaced + fixed the provisioning bug before any real customer hit it.

---

### Flow redesign — contact/budget first, gap-gate, no approval (DEPLOYED `a33e358`)

Reworked the whole client flow to stop churn and guarantee a gap-free brief. New status flow: **`contact → budget → intake → finalizing → complete`**.

- **Contact first** (email pre-filled from sign-in, editable), then **budget** (client picks a range; AI re-matches the tier from the conversation *after* the intake), then the **intake conversation last**. Lead captured up front ⇒ a churn mid/post-intake no longer loses the lead. Sign-on wall KEPT (fraud protection).
- **Approval/review step removed.** When the intake finishes, the server generates brief+summary, sends both emails, builds provisioning, marks complete — all via Next `after()` so it survives the client closing the tab.
- **Gap gate (core of the product):** the model no longer decides "done". When it proposes completion, the server runs a holistic gap audit over the transcript; any build-blocking open question/assumption becomes the next question and the intake continues — only a clean audit wraps up (60-message cap backstop). Per-field extraction proved too flaky to gate on, so the audit is the gate.
- New `lib/complete-interview.ts`: `auditGaps`, `generateGapQuestion`, `advanceToFinalizing`, `finalizeAndSend`. Dormant now: approve/finalize/reopen routes + ReviewStep.
- **Tested on dev:** stages, gap-gate asking real follow-ups (caught missing exception-handling + approval rules on an "already complete" transcript), completion → `after(finalizeAndSend)` → emails + 10 provisioning items, tier refine small→medium. Confirmed live on prod (new interviews start at `contact`).
- **Not yet browser-verified on prod:** SummaryView polling UI + the message-route auto-complete branch (components individually verified). Worth a real browser walkthrough.

## Session 9 — 14 June 2026

### Goal: get the redesigned flow truly working end-to-end (it wasn't). DONE ✅

Started from the dashboard showing 4 interviews incl. an empty shell + a real client whose intake never finished. Ended with a verified, clean end-to-end flow and a deduped DB. Four fixes shipped, all `tsc`/`eslint` clean and deployed to prod.

**1. No more empty-shell "spam" interviews — deferred row creation (commit `18c8f1d`).**
A row used to be inserted at `status='contact'` the instant the agency clicked **New Interview**, so every generated-but-unused or abandoned-before-contact link left a permanent empty row in the DB/dashboard. Now:
- `api/interview/start` returns a bare UUID, **writes nothing**.
- `interview/[id]/page.tsx` renders the contact step when **no row exists yet** (uses `.maybeSingle()`).
- `api/interview/[id]/lead` **creates** the interview row on contact submit (FK-safe: interview before lead), going straight to `budget`; 409s if a row already exists past `contact`.
- Net: a row appears only once a client submits real contact details. Dashboard mirrors the DB with **no filtering** (user explicitly wanted DB-level, not a UI filter).

**2. Link is single-use after finishing — verified.** Guard matrix confirmed: `message` needs `intake`, `lead` needs/creates `contact`, `budget` needs `budget`, `reopen` needs `review` (never reached in new flow). A `complete` link only ever renders the read-only SummaryView. No path re-opens a finished interview.

**3. ROOT BUG — gap-gate infinite loop (commit `8b53024`).** First real test (Lia, 25 user turns, every field captured) never completed: she said she'd supply her Bit/Paybox payment URL later, and `auditGaps` flagged that missing URL as build-blocking **every turn** → asked forever → no finalize, no emails, no summary, and the client could keep talking (bounded only by the 60-msg cap). Fix: rewrote the audit prompt so a **deliverable the client has promised to provide later** (URL/credential/asset/account) is a known open dependency, **not** a blocking unknown — plus excludes agency-decidable implementation choices and already-answered points. Verified both ways before shipping: Lia's real transcript → **0 gaps (auto-completes)**; a deliberately thin intake → **5 gaps (keeps asking)**.

**4. Client no longer sees an AI-inflated budget (commit `8972373`).** `advanceToFinalizing` used to re-classify the tier via the AI and **overwrite** `tier/budget_min/budget_max` — which the client sees in BOTH the on-screen summary ("Your budget range") and the summary email. A simple project bumped Starter→Large, so the client saw $25k–50k after picking $1k–5k. Fix: `advanceToFinalizing` now only transitions to `finalizing`; the client's picked tier/budget is kept exactly as selected everywhere. Removed the now-dead tier re-match + transcript field-extraction in `message`/`advance` routes (`getMatchedTier` now unused, left in `pricing.ts` for a future internal-only agency hint). Also reworded the completion screen to: *"a full copy of this summary is on its way to the email you provided. Suki Systems will take it from here, and we'll only reach back out if a question comes up or we need to confirm an assumption before building."*

**Recovery + live verification.**
- Recovered Lia's stuck interview by POSTing `/advance` on prod → finalized, brief 16.8k, summary 2.8k, 14 provisioning items, both emails sent.
- **Test #2 (fresh link, full browser walkthrough): PASSED.** Auto-completed on its own, tier stayed `starter` as picked (budget fix confirmed), both emails delivered, dashboard row populated. User: *"working beautiful."*

**DB cleanup.** Deleted the 3 test rows (2 Lia smoke tests + the empty shell) via cascade. **3 real clients remain:** United/Metroll, Invictus Institute, AB Residential — all `complete`.

**Status: the contact → budget → intake → finalizing → complete flow now works end-to-end, verified in the browser on prod.**

---

## Next Session Priorities

**Core flow is DONE + verified end-to-end on prod (session 9).** Remaining:

1. **M9 test #10** — manual: go to `/dashboard/settings`, enter a BYO Anthropic key, run a new interview, confirm it uses that key
2. **M9 test #11** — manual: copy the agency brief from dashboard, paste into a Claude Code session, confirm it generates a working project scaffold
3. **(Optional) Internal-only AI tier hint** — session 9 removed the AI tier re-classification entirely because it was overwriting (and showing the client) an inflated budget. If the agency still wants the AI's complexity read, re-add it as an internal field that NEVER changes what the client sees. `getMatchedTier` is still in `lib/pricing.ts`, unused, ready for this.
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
