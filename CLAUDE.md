# eGet-A-Room — Prototype Rebuild

## What this is
A rebuild of eGet-A-Room, a South African rental marketplace ("Find It. Rent It. Manage It.").
The original was generated with an AI builder (Hercules, live at https://eget-aroom-ai.onhercules.app).
We are rebuilding it as a **demo prototype** to show the client, then iterate on his feedback.
The full review of the original site (pages, forms, statuses, issues) is in `docs/site-review.md` — read it before building any feature.

Priority: a working, good-looking demo fast. Not production hardening.

## Stack (do not swap without asking)
- **Frontend:** Next.js (App Router) + TypeScript + Tailwind CSS + shadcn/ui + lucide-react icons
- **Backend:** Supabase — Postgres, Auth (Google OAuth), Storage, Realtime, Row Level Security
- **Supabase client:** `@supabase/ssr` for server components / route handlers / middleware
- **Maps:** Leaflet + react-leaflet + OpenStreetMap tiles (client component, dynamic import with ssr:false)
- **AI:** Next.js route handler calling an LLM API (key in `.env.local`, never in client code)
- **Hosting:** Vercel (frontend), Supabase cloud (backend)
- **Package manager:** npm

## Environment variables (`.env.local`, never commit)
```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=   # server-only, only for seed scripts
AI_API_KEY=                  # server-only
```
Never print, log or hard-code keys. Never use the service role key in browser code.

## Design system (match the original)
- Dark theme only. Font: **Plus Jakarta Sans** (next/font/google).
- Colours (Tailwind CSS variables, oklch):
  - background `oklch(14% .03 245)` (deep navy)
  - card `oklch(19% .04 245)`, muted `oklch(25% .04 245)`, accent `oklch(28% .07 245)`
  - primary `oklch(62% .15 220)` (cyan/sky), primary-foreground `oklch(10% .02 240)`
  - foreground `oklch(95% .01 240)`, border `oklch(100% 0 0 / .1)`
  - yellow accent for the second line of the hero headline
- Radius `0.75rem`. Rounded cards with subtle borders, icon tiles with tinted backgrounds.
- Currency is ZAR, formatted `R12,000/mo` (en-ZA).
- Mobile-first. Dashboards: left sidebar on desktop, bottom nav + "More" sheet on mobile.

## Roles (prototype)
`tenant`, `landlord`, `admin`. (Estate Agent / Property Manager / Maintenance Provider are out of scope for v1 — show them on the role picker as "Coming soon", disabled.)
- Role is chosen **once** at `/onboarding` after first sign-in. After that only an admin can change it.
- Enforce roles in the database with RLS, not just in the UI.
- Route groups: `/tenant/*`, `/landlord/*`, `/admin/*`, guarded in middleware; wrong role → redirect to own dashboard; no role → `/onboarding`.
- First admin: set via SQL seed (`update profiles set role='admin' where email=...`). Document this in README.

## Data model (Postgres, snake_case)
- `profiles` (id = auth.users.id, full_name, email, phone, avatar_url, role, agency_name, bio, is_verified, is_suspended, notification_prefs jsonb, created_at)