# Deployment Status — Live Debugging Session
_Last updated: 2 June 2026_

## Vercel Deployment
- **URL:** `intake-engine-881jqdx0l-suki-systems.vercel.app`
- **Status:** Live and deployed ✅
- **Repo:** `giltruman1906-byte/intake-engine` (private) → main branch
- **Latest commit:** `d4adf1a` — fixed admin Supabase client (was causing 404 for clients)

## Agency Login (Dashboard)
- **URL:** `intake-engine-881jqdx0l-suki-systems.vercel.app/login`
- **Email:** `giltruman1906@gmail.com`
- **Password:** stored in your head — created via Supabase Admin API ✅

## What is Working ✅
- Agency can log in to dashboard
- "New Interview →" button creates a fresh interview and shows a shareable link (Copy button)
- Client opens link → hits sign-up page at `/interview/{id}/auth`
- Client signs up with email + password
- Full interview flow: chat → budget → contact form → summary screen
- Admin Supabase client fixed (was causing 404 — now uses raw supabase-js)

## What is Broken / In Progress ❌

### 1. Email delivery — NOT working yet
**Root cause:** `suki-systems.com` domain not verified in Resend.
Emails are sent `from: intake@suki-systems.com` — Resend silently drops them without domain verification.

**DNS records needed in Squarespace (Custom Records section):**

| Type | Host | Data | Priority |
|------|------|------|----------|
| TXT | `resend._domainkey` | `p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC/7Z/bgx2KRThyAiSOHqQoItwoJhT9kpGVVb4O5kUso/aOGG3B4jXBbllSuQdjPMXBRhMbxqUry7gar3nmpb1fCLBAZq+dfullMZQPEkXwwXDXHCG+MoJt3i1ALmI2tWvPcIkxMotcRtp9pnps+ib9+o/FBP9ckBrlIjSIjlLEwQIDAQAB` | — |
| MX | `send` | `feedback-smtp.us-east-1.amazonses.com` | `10` |
| TXT | `send` | `v=spf1 include:amazonses.com ~all` | — |

**Steps remaining:**
1. Go to squarespace.com → Domains → suki-systems.com → DNS Settings → **Custom Records**
2. Add the 3 records above
3. Go back to Resend → Domains → suki-systems.com → click **"Verify DNS Records"**
4. DNS propagation: 5–30 minutes, may take up to 24h

**Note:** DKIM and SPF rows in Resend already show green checkmarks — may already be partially verified. Try Verify first before adding records.

### 2. Dashboard showing empty interviews
**Symptom:** Dashboard interview list shows no rows even after client completed the flow.
**Likely cause:** The interview that was tested was created before the admin client fix — it may have a corrupted state. OR the RLS policy on the `interviews` table is blocking the dashboard query (dashboard uses `createServerSupabaseClient` with anon key + user session, which is subject to RLS).
**Next step:** After emails are working, create a fresh interview end-to-end and check if it appears in the dashboard.

### 3. Brief not received at yali@suki-systems.com
**Cause:** Same as email issue — Resend domain not verified.
**Also check:** The `finalize` route is called by `SummaryView` on mount. Confirm the client actually saw the summary screen (not just the contact form). If they did, finalize ran and the brief is in the `briefs` table — it just wasn't emailed.

## Testing Order (once DNS is done)
1. ✅ Verify DNS in Squarespace
2. ✅ Click "Verify DNS Records" in Resend
3. Create a brand new interview from dashboard
4. Go through the full flow as a test client (use a real email to confirm receipt)
5. Check dashboard → interview detail for provisioning checklist
6. Confirm brief arrives at yali@suki-systems.com
7. Confirm client summary arrives at client email

## Environment Variables (Vercel — all set ✅)
| Variable | Status |
|---|---|
| ANTHROPIC_API_KEY | ✅ |
| ANTHROPIC_MODEL | ✅ |
| NEXT_PUBLIC_SUPABASE_URL | ✅ |
| NEXT_PUBLIC_SUPABASE_ANON_KEY | ✅ |
| SUPABASE_SERVICE_ROLE_KEY | ✅ |
| RESEND_API_KEY | ✅ |
| BRIEF_DELIVERY_EMAIL | ✅ `yali@suki-systems.com` |
| ENCRYPTION_KEY | ✅ |
| SESSION_SECRET | ✅ |

## Supabase
- **Project URL:** `https://wifvirdxegyyflcrzlsf.supabase.co`
- **Email confirmation:** OFF ✅ (clients don't need to verify email before signing in)
- **Agency #1:** Suki Systems — UUID `00000000-0000-0000-0000-000000000001`

## Resend
- **Account:** giltruman1906@gmail.com
- **Domain:** suki-systems.com — NOT yet verified ❌
- **API key:** set in Vercel env vars ✅
