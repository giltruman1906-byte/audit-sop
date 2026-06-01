# BUILD_BRIEF.md — Intake Engine (working name)

> **Read me first, Claude Code.** This is your single source of truth. Build the milestones in
> §5 in order. After each one, run its done-criteria (§11) and report status before continuing.
> Do not guess on anything touching auth, API keys, secrets, billing, PII, or deletion — stop and ask.

---

## 0. What we are building (and what we are NOT)

A conversational web app that **interviews an agency's client about a process they want built or
automated, then produces two things on submit: (1) a Claude-Code-ready Markdown build brief
emailed to the agency, and (2) a plain-language summary shown to the client** (stack, what we'll
build, indicative timeframe, and the budget range they signed off on).

**It is an articulation aid, not a diagnostic.** The client usually already knows what they want —
they just can't explain it in a buildable way. The engine draws it out and structures it. It does
**not** decide what's worth doing or score ROI.

**The end-to-end flow:**
```
Interview (text/voice)  →  Budget step (pick a range from the agency rate card)
  →  Contact popup (company, name, email, phone)  →  Submit  →  TWO outputs
```

**Three roles:**
- **Agency** (Suki = tenant #1, our own beta): the SaaS customer. Logs in, supplies its own Claude
  API key, configures its rate card, starts an interview, shares a link with its client.
- **Client**: answers the questionnaire, picks a budget range, fills contact info, hits Submit.
- **Outputs**: full technical brief → agency (email); plain-language summary → client (on screen,
  optionally emailed).

This is the *Phase 1 blueprint* of our delivery methodology, run as a client-facing conversation,
producing both an internal handoff doc and a client-facing one.

---

## 1. Requirements Summary

| Field | Value |
|---|---|
| **Product** | Intake Engine (working name) |
| **Beta tenant** | Suki Systems (agency_id #1), running it with its own clients |
| **Job to be done** | Help a client articulate a process they understand but can't explain clearly |
| **NOT in the job** | ROI scoring, "should you automate this", holistic auditing |
| **Input** | Conversational — text or voice |
| **Budget** | Client picks a range from the **agency's configured rate card** (not LLM-invented); signs off; added to summary + brief |
| **Lead capture** | On submit: company name, contact name, email, phone (popup) |
| **Outputs** | (a) Full technical MD brief → agency email · (b) Plain-language summary → client |
| **Engine** | Claude API (schema-constrained interview + brief + summary generation) |
| **Stack** | Claude API · Next.js on Vercel · Supabase (DB + Auth + secret storage) · Resend (email) |
| **SaaS model** | Multi-agency; each agency brings its own Claude (later OpenAI) key + own rate card |
| **Constraints** | Secrets never in doc/email/logs; never auto-create accounts; keys encrypted; estimates are indicative, not binding quotes; contact PII stored securely with stated purpose |

---

## 2. Stack (and why each box exists)

```
RECOMMENDED STACK
  Engine     : Claude API (Anthropic) — interview + brief + client summary
  UI + API   : Next.js (App Router) on Vercel — chat UI AND serverless API routes, one project
  Data       : Supabase — Postgres (sessions, transcripts, briefs, leads, rate cards)
               + Auth (agency login) + encrypted storage for per-tenant API keys
  Email      : Resend — brief to agency; summary to client (optional)
  OPTIONAL   : Voice via browser Web Speech API (free) → transcription API later
               Provider abstraction so OpenAI is a config swap, not a rewrite
```

**Why not "just Claude"?** Claude runs the conversation, but you need Supabase to hold sessions,
briefs, leads, and the encrypted BYO keys, and Resend to deliver outputs. Bare-MVP could skip the
DB, but it breaks the moment a second agency signs up.

**Trade-offs:** Vercel functions have time limits — stream the chat and keep generation under the
limit (or background it). Browser voice is free but flaky — fine for v1.

**Estimated effort:** ~7–10 working days for the full beta (the budget + contact + dual-output add
~2 days over the prior scope).

---

## 3. Architecture (end to end)

```
 CLIENT (browser, text or voice)
        │ message
        ▼
 Next.js chat UI ──► POST /api/interview/{id}/message ──► Claude (interview spec §6)
        │ ... loop until engine signals "enough to fill the schema" ...
        ▼
 BUDGET STEP  ──► engine maps captured scope → tier ──► shows agency rate-card range
        │ client selects + signs off a range (stored)
        ▼
 CONTACT POPUP ──► company, name, email, phone  ──► stored in `leads`
        │ Submit
        ▼  POST /api/interview/{id}/finalize
   Claude generates, schema-constrained:
     ├─ AGENCY BRIEF (full technical, §7a) ──► Supabase + Resend ──► agency inbox
     └─ CLIENT SUMMARY (plain language, §7b) ──► Supabase + shown on screen
                                              └─ optionally Resend ──► client email
        │
        ▼
 Agency drops the brief into Claude Code → builds the project
```

---

## 4. Repo structure

```
intake-engine/
├── README.md
├── .env.example
├── app/
│   ├── (auth)/login/page.tsx
│   ├── dashboard/
│   │   ├── page.tsx                # list interviews, view briefs/leads
│   │   └── rate-card/page.tsx      # agency configures pricing tiers (seed Suki first)
│   ├── interview/[id]/
│   │   ├── page.tsx                # chat UI (text + voice toggle)
│   │   ├── BudgetStep.tsx          # tier-mapped range + sign-off
│   │   ├── ContactPopup.tsx        # company/name/email/phone
│   │   └── SummaryView.tsx         # client-facing summary after submit
│   └── api/
│       ├── interview/start/route.ts
│       ├── interview/[id]/message/route.ts
│       ├── interview/[id]/voice/route.ts          # optional
│       ├── interview/[id]/budget/route.ts          # map scope→tier, save sign-off
│       ├── interview/[id]/lead/route.ts            # save contact info
│       ├── interview/[id]/finalize/route.ts        # generate BOTH outputs + email
│       └── interview/[id]/outputs/route.ts         # fetch brief + summary
├── lib/
│   ├── claude.ts                   # provider-abstracted LLM client
│   ├── interview-spec.ts           # interview prompt + schema (CORE IP, §6)
│   ├── brief-generator.ts          # transcript → agency brief (§7a)
│   ├── summary-generator.ts        # transcript + budget → client summary (§7b)
│   ├── pricing.ts                  # scope → tier mapping against rate card
│   ├── crypto.ts                   # encrypt/decrypt tenant API keys
│   ├── email.ts                    # Resend wrapper
│   └── supabase.ts
├── db/schema.sql                   # tables + RLS (§7c)
└── tests/
    ├── interview-spec.test.ts
    ├── pricing.test.ts             # scope→tier deterministic mapping
    ├── brief-generator.test.ts
    └── summary-generator.test.ts
```

### .env.example
```
# Claude (Anthropic)
ANTHROPIC_API_KEY=            # Suki's key for beta; per-tenant + encrypted for SaaS
ANTHROPIC_MODEL=
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=    # SERVER ONLY — never expose to client
# Email
RESEND_API_KEY=
BRIEF_DELIVERY_EMAIL=         # agency inbox that receives finished briefs
# App
ENCRYPTION_KEY=               # 32-byte key for encrypting tenant API keys at rest
SESSION_SECRET=
```

---

## 5. Build order (milestones)

- **M0 — Scaffold.** Next.js + Vercel + Supabase + Auth. Seed Suki as agency #1 with a starter rate card. Tables + RLS applied.
- **M1 — Interview loop (the heart).** `interview-spec.ts`; `/message` streams Claude follow-ups; transcript persists. Get the leading-question behaviour right before anything else.
- **M2 — Budget step.** `pricing.ts` maps captured scope → a tier; `BudgetStep.tsx` shows the agency's rate-card range; client selects + signs off; stored on the interview.
- **M3 — Contact capture.** `ContactPopup.tsx` + `/lead` → save company/name/email/phone to `leads`, with a one-line purpose statement shown to the client.
- **M4 — Finalize → dual output.** `/finalize` generates BOTH the agency brief (§7a) and the client summary (§7b), schema-constrained; store both; render `SummaryView.tsx`.
- **M5 — Email delivery.** Resend: brief → agency inbox; summary → client email (optional toggle). No secrets in either (see §8).
- **M6 — Provisioning + vault handoff.** Generate `provisioning_items`; guide the client to create accounts; secrets to a vault, brief references names only.
- **M7 — BYO key + multi-tenant + rate-card config UI.** Per-tenant encrypted Claude key; `rate-card/page.tsx`; RLS verified across two test agencies.
- **M8 — Voice (optional).** Browser Web Speech API → text → same `/message` path.
- **M9 — Polish + acceptance.** Resume sessions, edit brief/summary before send, run §11.

---

## 6. The interview engine (CORE IP)

The engine fills a **known schema by conversation.** System prompt (`interview-spec.ts`) encodes:
- **Stance:** articulation aid; assume the client knows what they want; make it sayable and buildable; never judge whether it's worth doing.
- **Behaviour:** one leading question at a time, plain language, follow up on vagueness, draw out details clients forget.
- **Fields to extract:** `project_name · what_they_want · current_process_steps · data_sources ·
  where_data_lives · trigger_and_frequency · inputs · outputs · who_uses_output ·
  integrations_needed · constraints · success_criteria · edge_cases · rough_volume ·
  complexity_signals` (the last feeds the budget tier in §7d).
- **Stop condition:** stop only when every required field is filled well enough to build from.
- **No diagnosis, no pricing in chat:** budget is a separate, rate-card-driven step (§7d), not a number the model invents mid-conversation.

This prompt + schema is the product's defensibility. Keep it the single editable artifact.

---

## 7. Outputs, budget logic, and DB

### 7a. Agency brief (full technical — internal)
```
# <project_name>
## Summary · ## Requirements (extracted fields) · ## Recommended stack (+ rationale, trade-offs)
## Architecture (text data-flow) · ## Build plan (folders + key files)
## Provisioning checklist (accounts + env var NAMES + vault locations — NO values, §8)
## Estimated timeframe + budget tier (indicative) · ## Client contact (from `leads`)
## Acceptance criteria · ## Open questions / assumptions
```

### 7b. Client summary (plain language — external, 1 page)
```
What we understood   — the process in their words, no jargon
What we'll build      — plain-language description of the solution
The tools involved    — stack explained simply ("a database to store X, an AI step to do Y")
Indicative timeframe  — a range, clearly labeled an estimate
Your budget range     — the range they signed off on
Next step             — "[Agency] will confirm scope and final pricing and reach out"
```
> The summary must state it is an **indicative estimate, with final scope and pricing confirmed by
> the agency** — never phrased as a binding quote.

### 7c. DB schema (Supabase, RLS-scoped by `agency_id`)
```sql
agencies(id, name, created_at)                                       -- row 1 = Suki
agency_credentials(agency_id, provider, key_ciphertext, created_at)  -- encrypted BYO keys
pricing_tiers(id, agency_id, tier, label, price_min, price_max,
              typical_timeframe, complexity_hint)                    -- the rate card
interviews(id, agency_id, client_label, status, tier, budget_min,
           budget_max, estimated_timeframe, created_at, completed_at)
messages(id, interview_id, role, content, ts)                        -- transcript
leads(id, interview_id, company_name, contact_name, email, phone, consent_ts, created_at)
briefs(id, interview_id, agency_markdown, client_summary_markdown,
       schema_json, created_at)
provisioning_items(id, interview_id, service, env_var_name, vault_location, status)
```

### 7d. Budget logic (rate-card driven — do NOT let the model invent prices)
- The agency configures `pricing_tiers` once (Small/Medium/Large or similar: label, price range,
  typical timeframe, and a `complexity_hint` describing what fits the tier).
- `pricing.ts` maps the interview's `complexity_signals` to a tier **deterministically** where
  possible (rules), using the model only to classify ambiguous cases against the tier hints.
- The client is shown the matched tier's range, can adjust to an adjacent range, and **signs off**.
  The chosen `tier / budget_min / budget_max / estimated_timeframe` save to `interviews`.
- A budget that's far below the scope tier is useful signal — surface it to the agency in the
  brief ("client budget below typical range for this scope").

---

## 8. Provisioning + secrets handoff

The engine **guides the client to create accounts themselves** — never creates accounts for them.
Secret values **never enter the brief, the summary, the email, or any log.** Client creates the
account → pastes the secret **directly into a vault**; the brief lists secrets by **name + location
only** (`SUPABASE_SERVICE_KEY= # in vault, do not commit`). Brief travels by email (no secrets);
secrets travel to the vault. This is also a selling point: "we never touch your keys."

---

## 9. BYO API key handling (security)

Encrypted at rest (`agency_credentials.key_ciphertext`), decrypted **server-side only** per
request, never sent to the browser, never logged. Provider-abstracted (`lib/claude.ts`) for later
OpenAI support. Single env key is fine for the Suki beta, but build the encrypted-per-tenant path
now so SaaS onboarding isn't a retrofit.

---

## 10. Guardrails (non-negotiable)

1. **No secret ever in a brief, summary, email, transcript, or log.** Names + vault locations only.
2. **Never auto-create accounts** for a client — guide them to do it themselves.
3. **API keys encrypted at rest, decrypted server-side only, never logged.**
4. **Service-role Supabase key is server-only** — never in the client bundle.
5. **Budget = rate-card driven, never LLM-invented**, and every estimate is labeled **indicative,
   not a binding quote.**
6. **Contact PII** (`leads`) stored securely, shown a clear purpose statement + consent timestamp;
   collect only company/name/email/phone, nothing more.
7. **Stop and ask** before anything touching auth, billing, key handling, PII, or deletion.
8. **Schema-constrained output:** both brief and summary must match §7 exactly; validate before
   storing/sending.
9. **Tests as you go.**

---

## 11. Acceptance tests — all must pass before the Suki beta runs a real client

```
[ ] 1. Agency logs in; sees only its own interviews/leads (RLS verified with 2 agencies).
[ ] 2. Interview asks ONE leading question at a time; follows up on vague answers.
[ ] 3. Engine stops only when every required field is filled; transcript persisted.
[ ] 4. Budget step shows the AGENCY's rate-card range (not invented); scope→tier mapping is consistent on re-run; sign-off saves tier + range + timeframe.
[ ] 5. Contact popup captures company/name/email/phone with a purpose statement; saved to `leads` with consent_ts.
[ ] 6. Submit produces BOTH outputs: agency brief (§7a) + client summary (§7b), each matching schema.
[ ] 7. Brief emailed to agency; summary shown to client (and emailed if toggled). Neither contains any secret value.
[ ] 8. Client summary labels timeframe + budget as INDICATIVE, not a quote.
[ ] 9. Provisioning checklist lists accounts + env var NAMES + vault locations, no values.
[ ] 10. BYO key stored encrypted; decrypt works server-side; key never in client or logs.
[ ] 11. Drop a generated brief into Claude Code → it scaffolds the project without clarification.
```

Test #11 is the real one: the brief is only good if Claude Code can build from it unaided.

---

## 12. Out of scope (do not build now)

Billing/Stripe · agency self-serve signup · OpenAI provider (stub the abstraction only) ·
managed-key tier · the project-building itself (that's Claude Code on the agency side) · any
ROI/diagnostic scoring · the desktop activity agent (separate, later, premium add-on) · CRM
integration for leads (export/email is enough for v1).

---

## 13. How to work, Claude Code

- Build §5 in order; **M1 (interview behaviour) is the heart** — nail it before polish.
- Keep `interview-spec.ts` and `pricing.ts` as the two pieces of real IP; don't improvise them.
- After each milestone: run its done-criteria, commit, report status. Don't chain silently.
- Re-check §10 on every change touching keys, secrets, PII, the brief, the summary, or email.
- Surface blockers immediately. Stop and ask before auth, key handling, billing, PII, or deletion.
