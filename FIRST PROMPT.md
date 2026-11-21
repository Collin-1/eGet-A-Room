# First prompt for Claude Code

Paste this after you have:
1. Created an empty folder and put `CLAUDE.md` in its root.
2. Exported the review doc (Download → Markdown) and saved it as `docs/site-review.md`.
3. Created a Supabase project yourself (free plan, closest region) and a GitHub repo.
4. Connected the Supabase MCP server to Claude Code (see the setup steps below the prompt).

---

```
Read CLAUDE.md and docs/site-review.md fully before doing anything.

We are building a prototype rebuild of eGet-A-Room to demo to a client. Follow the stack, design system, data model and scope order in CLAUDE.md exactly.

Start with scope step 1 only:
- Scaffold a Next.js App Router + TypeScript + Tailwind + shadcn/ui project in this folder.
- Set up the dark theme and Plus Jakarta Sans exactly as specified.
- Set up Supabase clients with @supabase/ssr, and middleware for auth + role-based route guards.
- Create the first migration: profiles table, role enum, trigger to create a profile on sign-up, RLS policies (a user can read/update their own profile but NOT their role once set; admins can update anyone's role).
- Google sign-in, /auth/callback, and the /onboarding role picker (Tenant, Landlord enabled; Estate Agent, Property Manager, Maintenance Provider shown as "Coming soon").
- Empty dashboard shells for /tenant, /landlord and /admin with the sidebar layout.

Before writing code, show me a short plan and the list of files you'll create. Tell me exactly which values I need to put in .env.local and what I need to configure in the Supabase dashboard for Google sign-in (I'll do those clicks myself). Then build it, run lint and build, and tell me how to test it.
```

---

## Connecting Claude Code to Supabase (no browser needed)

In your project folder, run:

```
claude mcp add --transport http supabase https://mcp.supabase.com/mcp
```

Then start Claude Code, run `/mcp`, pick **supabase** and sign in to Supabase in the browser window that opens. This lets Claude Code create tables, run migrations and check data in your project directly.

Safety tips:
- Use a separate **dev** Supabase project for this, not anything holding real data.
- In Supabase's MCP setup you can scope the connection to one project and make it read-only; turn read-only off only while it's applying migrations.
- Keep the service role key and AI key in `.env.local` only. Don't paste them into chat.

## Prompts for the next steps (one at a time)

- "Scope step 2 from CLAUDE.md: the public site. Seed the 7 listings with placeholder photos first."
- "Scope step 3: landlord properties."
- "Scope step 4: admin approval and users. Also add the SQL in README for making my account the first admin."
- …and so on through step 8. Test each one before moving on.