# eGet-A-Room — Site Review & Rebuild Spec

Nov 20, 2026 · @Collin Makwala

## Summary

eGet-A-Room is a working, reasonably complete rental marketplace for South Africa: tenants search and apply, landlords list and manage, admins approve. It is not a mock-up. About 40 backend functions are wired to a real database, and the AI search and AI listing writer both work.

It was generated on Hercules (a no-code AI app builder) and shipped as one 1.6 MB JavaScript bundle on `eget-aroom-ai.onhercules.app`. The rebuild is mostly about ownership, SEO, compliance and a few missing flows, not inventing the product. Use this doc as the spec: every page, form, status and backend call below was observed on the live site or read from its bundle.

**How this was reviewed:** I browsed every public page, then signed in as `Words Words` and switched the account through Tenant, Estate Agent, Maintenance Provider, Landlord and Property Manager via `/onboarding`, checking each dashboard live. The account was left on Property Manager, its original role. Admin cannot be chosen at onboarding, so the Admin panel is reconstructed from the UI text and function calls in the bundle. I opened forms (viewing, application, maintenance) but submitted none, and created no listings or messages.

## Tech stack found in the current build

| Layer | What it uses now | Notes for the rebuild |
| --- | --- | --- |
| Frontend | React SPA, Vite build, Tailwind + shadcn/ui-style components, Framer Motion | Client-side rendering only, so listings are not indexable by Google |
| Backend / DB | Convex (real-time queries, mutations, file storage via `generateUploadUrl`) | All data access is through named Convex functions (inventory below) |
| Auth | Hercules-hosted OIDC login (email shown as "Managed by your login provider") | You will need your own auth (Clerk, Auth0, ASP.NET Identity, etc.) |
| Maps | Leaflet + OpenStreetMap tiles on the property page | Fine to keep; add geocoding for addresses |
| AI | `ai.generateListing` (listing description + social post) and an LLM-backed natural-language search | Model/provider not visible from the client |
| Payments | None live. Rent is recorded manually; Stripe code appears in the bundle but is not used in any visible flow | For SA, consider PayFast, Paystack, Ozow or Peach |
| Hosting | Hercules subdomain with a "Built with Hercules" badge on every page | Needs a real domain (egetaroom.co.za is referenced in the footer email) |

**Design tokens:** dark-only theme. Font is Plus Jakarta Sans. Background `oklch(14% .03 245)` (deep navy), cards `oklch(19% .04 245)`, primary `oklch(62% .15 220)` (cyan/sky blue), a yellow accent on the hero headline, radius `0.75rem`, borders white at 10% opacity. Meta description: "South Africa's AI-powered property rental platform."

## Roles and access model

Onboarding offers five roles, but the app only has three dashboards. Three of the five roles all land on the same landlord dashboard.

| Role picked at `/onboarding` | What actually happens (tested) | Gap |
| --- | --- | --- |
| Tenant | `/tenant/*` dashboard, 11 pages | None |
| Landlord / Property Owner | `/landlord/*`, 11 pages. Settings has no Agency name field | None |
| Estate Agent | Identical `/landlord/*` dashboard; Settings adds Agency name | No link between an agent and the owners they list for |
| Property Manager | Identical to Estate Agent | No portfolio or owner reporting |
| Maintenance Provider | **Blank page.** Redirected to `/landlord`, which renders nothing but the Hercules badge; `/tenant` also bounces there | Role is broken: the user is stuck with no way in except revisiting `/onboarding` |
| Admin (not selectable) | `/admin/*` | Not observed. Likely set via "Change Role" in the admin panel, with the first admin set directly in the database (unconfirmed; ask the builder) |

Routes are role-guarded: visiting `/tenant` or `/admin` as a landlord silently redirects to `/landlord`. Unauthenticated users hitting protected screens see "Please sign in to continue". Role is stored on the user via `users.setRole` and can be changed later only by an admin.

## Public site pages

The public site has six pages plus a 404. Shared header: logo, Home, Search, List Property, How It Works, About, and "My Dashboard" + "Sign Out" when signed in. Shared footer: tagline, quick links, Johannesburg address, enquiries@egetaroom.co.za, +27 72 489 8057, six social links, and Privacy Policy / Terms links that go nowhere (`#`).

| Route | What is on it |
| --- | --- |
| `/` Home | Hero ("Find Your Perfect Home Today", Cape Town photo), quick search bar (location, property type, max budget, bedrooms). Stats band (properties, tenants, owners, cities) that shows "—" for every value. Featured Properties carousel, Browse by Type tiles (Room, Backroom, Cottage, Apartment, House, Flat), More Listings, 4-step How It Works, three testimonials, landlord CTA with marketing stats (95% occupancy, 14 days to let) |
| `/properties` Search | Natural-language AI search box ("Try: affordable room in Katlehong under R2500"). Filters: city/suburb/township text, province, type, max rent, bedrooms, Pet Friendly and Furnished toggles. Sort (Newest first). Removable filter chips, "Clear filters", Load more. 7 listings live, all in Gauteng |
| `/properties/:id` Listing | Photo gallery with thumbnails + "View all photos", Verified and Available Now badges, Share. Title, location, rent, deposit, beds/baths/parking/type chips. Description, Property Details grid, Utilities & Inclusions, address, Leaflet map. Reviews (only tenants with a lease can review; landlord can respond). Sticky side card: Book a Viewing, Apply Now, Save Property, Contact Landlord. "Listed by" card, availability card, Similar Properties |
| `/how-it-works` | 4 tenant steps, 4 landlord steps, 4-question FAQ (free for tenants, how verification works, backrooms allowed, manual rent tracking) |
| `/about` | Story, the stats band again (also "—"), four values (Trust & Safety, Community First, Smart Technology, Housing Dignity), eight operating cities, CTA |
| `/list-property` | Landlord pitch, six benefit tiles, "Create New Listing" (or "Sign in to create your listing" when logged out) |
| `/onboarding` | Role picker, five cards, Continue |
| `*` | 404 "Page Not Found" with Return to Home |

**AI search, tested:** "furnished room in Pretoria under R4000" took about 5 seconds, then set city = Pretoria, province = Gauteng, type = Room, max rent = R4,000 and showed a one-line interpretation. It returned the one correct listing.

## Landlord / agent / manager dashboard (`/landlord/*`)

Eleven sidebar pages, all seen live (every one is empty for this account). Layout: left sidebar with avatar, name and role at the top; Public Site and Sign Out at the bottom. On mobile it collapses to a bottom nav with a "More" menu.

| Page | Contents and actions |
| --- | --- |
| Dashboard | Welcome line, Add Property. Four stat cards: Total Properties, New Applications, Active Leases, Open Maintenance. Recent Properties list, or an empty-state CTA |
| My Properties | Listing count, AI Listing Generator, Add Property. Cards with status badge (Draft, Pending Review, Published, Rented, Archived), Edit, Archive, Delete (with confirm) |
| Add / Edit Property (`properties/new`, `properties/:id/edit`) | Details: type, title, description. Location: province (9 SA provinces), municipality, city/town, township, suburb, street address. Pricing: monthly rent, deposit, optional cleaning fee, available-from date. Features: bedrooms, bathrooms, parking (+ optional parking cost), furnished, pet friendly. Utilities: electricity (Pre-paid / Metered / Included), water (Included / Metered), sewerage, refuse. Photos: at least 1, max 10 MB each, first is cover, set-as-cover, remove; only after the first save. Buttons: Save Draft, Submit for Review |
| AI Listing Generator (modal) | Title, type, rent, beds, baths, city, province, key features. Returns a listing description and a social media post, with Use This, Generate Again and Done |
| Applications | Tabs: All, New, Reviewing, Approved, Rejected. Per application: applicant details, notes, documents. Start Review; Approve (optional message; marks property Rented, creates lease, notifies tenant); Reject (reason required, shared with tenant) |
| Leases | List with Active, Expiring Soon, Expired, Terminated. Terminate Lease. Leases are created automatically on approval |
| Viewings | Tabs: All, Pending, Confirmed, Completed, Cancelled. Confirm, Complete, Cancel. Live location tracking for the viewing trip ("People en route", "Live Tracking & Directions") |
| Messages | Two-pane inbox: conversation list + thread. Started from "Contact Landlord" on a listing |
| Payments | Totals: Total Due, Received, Overdue. Tabs: All, Pending, Overdue, Partial, Paid. Create Payment (property, then tenant with an active lease, amount due, due date, description). Mark Overdue. Manual records only |
| Maintenance | Totals: Open Requests, Urgent. Tabs: All, New, Reviewing, In Progress, Completed. Status buttons (Start Review, Mark Assigned, Start Work, Mark Complete) with an optional note to the tenant |
| Notifications | Feed with Load more; empty state "You're all caught up" |
| Analytics | Eight KPI cards (properties, occupancy rate, rent collected + collection rate, rent due, pending applications, viewings, open maintenance, active leases). Charts: rent collection trend, application breakdown, viewing status, maintenance by category. Property Performance table (property, city, status, rent, apps, views, maint.) |
| Settings | Profile tab: photo (5 MB), account type, full name, email (read-only), phone, agency name, skills ("Plumbing, Electrical", for providers), about me, Save. Notifications tab: toggles for messages, applications, viewings, maintenance, payments |

## Tenant dashboard (`/tenant/*`)

Checked live with the account switched to Tenant. Sidebar: Dashboard, Find Property, Saved, Viewings, Applications, My Leases, Messages, Payments, Maintenance, Notifications, Settings.

| Page | Contents and actions (as seen) |
| --- | --- |
| Dashboard | "Welcome back" + four shortcut tiles (Find a Property, My Viewings, Applications, Saved). Cards: Active Applications, Saved Properties (each with View all), Rental Status ("No active lease"). Search Properties CTA |
| Find Property (`/tenant/search`) | City/suburb/township search + a Filters drawer (province, type, max rent, min bedrooms 1+/2+/3+). Card grid with a heart (save) on each card. **No AI search here**, unlike the public `/properties` page |
| Saved | "0 saved listings" count, grid, empty state with Browse Properties |
| Viewings | "My Viewings", count, Find Properties button. Statuses Requested, Confirmed, Completed, Cancelled; Cancel Viewing; share live location en route |
| Applications | Count, Find Properties. Statuses Submitted, Under Review, Approved, Rejected / Withdrawn; landlord note; Withdraw with confirm |
| Apply form (`/tenant/applications/new/:propertyId`) | Property summary card. Name and email prefilled from login. Phone, employment status (Employed, Self-employed, Unemployed, Student, Retired), monthly income, occupants (default 1), desired move-in date, notes. Documents: pick a type (ID/Passport, payslip, bank statement, employment letter, reference letter, Other) then Upload File; PDF, JPG, PNG, DOC, DOCX up to 10 MB. Submit Application |
| Book a Viewing (modal on listing) | Preferred date + preferred time (free choice, no landlord availability slots), optional notes, Request Viewing |
| My Leases | Current and past leases; Active / Expiring Soon / Expired / Terminated |
| Payments | Four totals: Total Due, Total Paid, Overdue, Pending. Tabs All, Pending, Overdue, Partial, Paid. Payments only appear once the landlord creates them; tenant then uses Record Payment (amount, bank reference, notes) |
| Maintenance | Tabs All, New, Reviewing, In Progress, Completed; New Request. Without an active lease the modal blocks submission ("requests can only be submitted for properties you are currently renting"). With a lease: property, category, priority, title, description, up to 5 photos; detail view with comment thread |
| Messages, Notifications | Same as landlord |
| Settings | Same as landlord, minus Agency name |

## Admin panel (`/admin/*`)

Not seen live: no reachable account has the admin role, and visiting /admin as tenant or landlord redirects to /landlord. Everything below comes from the site's code: the /admin/\* route, the admin.\* and properties.admin\* backend functions, and the admin UI text. Whether it is used in production, and how admins are created, needs confirming with the builder. Tabs: Overview, Users, Properties, Applications, Maintenance, Analytics.

| Page | Contents and actions |
| --- | --- |
| Overview | Platform stats: total users, tenants, landlords/agents, suspended, overdue payments, rent tracked. "Require immediate attention" block. Shortcuts to Manage Users, Review Properties, Monitor Applications, Maintenance Oversight, Platform Analytics |
| Users | Filter by role (Tenants, Landlords, Estate Agents, Property Managers, Admins). User detail: role, joined, onboarding state, Change Role, verify, suspend |
| Properties | Pending queue (Approve / Reject) and All Properties. Approval is what makes a listing "Published" and "Verified" |
| Applications | Read-only view of every application: income, contacts, occupants, move-in, property rent, tenant and review notes |
| Maintenance | Every maintenance request on the platform |
| Analytics | User registrations and rent tracked over the last 6 months, properties by province, applications by status, property types |

## Data model and status lifecycles

Twelve entities cover the whole app. Field lists are inferred from forms and labels; the status values are exactly as shown in the UI.

| Entity | Key fields | Statuses |
| --- | --- | --- |
| User | name, email, phone, avatar, role, agency name, skills, bio, notification prefs, verified, suspended, onboarding done | — |
| Property | owner, type (Room, Backroom, Cottage, Apartment, House, Flat, Townhouse), title, description, province, municipality, city, township, suburb, address, lat/lng, rent, deposit, cleaning fee, parking + cost, beds, baths, furnished, pet friendly, electricity, water, sewerage, refuse, available from, verified | Draft → Pending Review → Published → Rented; Archived; Rejected by admin |
| PropertyImage | property, file, is primary, order | — |
| SavedProperty | tenant, property | — |
| Viewing | property, tenant, date/time, notes | Pending/Requested → Confirmed → Completed; Cancelled |
| LocationShare | viewing, user, live position | sharing / stopped |
| Application | property, tenant, personal + employment details, income, occupants, move-in date, notes, documents, review note | New/Submitted → Under Review → Approved or Rejected; Withdrawn |
| Lease | property, tenant, landlord, start/end, rent | Active, Expiring Soon, Expired, Terminated |
| RentPayment | lease, tenant, amount due, due date, description, amount paid, reference, notes | Pending, Partially Paid, Paid, Overdue |
| MaintenanceRequest + Update | property, tenant, category, priority, title, description, photos; comment thread | New → Reviewing → Assigned → In Progress → Completed |
| Conversation + Message | participants, property; sender, body, read | unread / read |
| Notification, Review | type, link, read; property, tenant, rating 1–5, text (min 10 chars), landlord response | — |

## Backend function inventory

These are the Convex functions the frontend calls, grouped by module. Treat each as an API endpoint to re-implement.

| Module | Functions |
| --- | --- |
| users | getCurrentUserQuery, updateCurrentUser, setRole, updateProfile, generateUploadUrl, removeAvatar, updateNotificationPreferences |
| properties | getFeaturedListings, getPublicStats, getById, getSimilar, listByOwner, create, update, submitForReview, archive, remove, listPending, adminListAll, adminApprove, adminReject |
| propertyImages | generateUploadUrl, addImage, removeImage, setPrimary |
| tenantQueries | getDashboardSummary, listPublished, isSaved, saveProperty, unsaveProperty, listSaved, submitApplication, listMyApplications, withdrawApplication, listNotifications, unreadNotificationCount, markNotificationRead, markAllNotificationsRead |
| applications | listApplications, markUnderReview, approveApplication, rejectApplication |
| viewings | requestViewing, listMyViewings, cancelViewing, listViewingsForLandlord, confirmViewing, completeViewing |
| tracking | updateLocation, stopSharing, getForContext, getPropertyLocation |
| leases | listTenantLeases, listLandlordLeases, terminateLease |
| rentPayments | createPayment, listLandlordPayments, markOverdue, landlordProperties, tenantsForProperty, listTenantPayments, tenantPaymentSummary, recordPayment |
| maintenance | submitRequest, generateUploadUrl, listTenantRequests, tenantLeaseProperties, getRequestDetail, addUpdate, listLandlordRequests, updateStatus |
| messaging | listConversations, getOrCreateConversation, getMessages, sendMessage, markAsRead, unreadCount |
| reviews | listByProperty, getAverageRating, canReview, submitReview, respondToReview |
| analytics / admin | analytics.getAnalytics; admin.stats, admin.users, admin.monitoring, admin.analytics |
| ai | generateListing (plus the NL search parser used on `/properties`) |

## Issues, gaps and risks

The biggest risks are legal and trust, not code: the site collects ID documents and bank statements with no privacy policy, and shows invented testimonials and stats.

| Priority | Issue | Where |
| --- | --- | --- |
| High | Choosing **Maintenance Provider** at onboarding leaves the user on a blank page with no navigation | `/landlord` |
| High | Any signed-in user can change their own role at any time by revisiting `/onboarding` (I switched one account through five roles). Role changes after signup should be admin-only or verified | `/onboarding`, `users.setRole` |
| High | Privacy Policy and Terms links point to `#`. The app collects IDs, payslips and bank statements and shares live location, so POPIA requires a privacy notice, consent and a retention policy | Footer, apply form |
| High | Testimonials ("Thabo M.", "Lindiwe K.", "Sipho N.") and claims like "South Africa's #1 PropTech Platform", "95% higher occupancy" and "14 days average" look AI-generated, with only 7 listings live. Replace with real data or remove | Home |
| High | No real payments: rent is self-reported by tenant or entered by landlord. The "Integrated rent payment tracking" pitch overstates this | Payments |
| High | Vendor lock-in: code, database and auth all live on Hercules/Convex under their subdomain | Whole app |
| Medium | Estate Agent and Property Manager get exactly the landlord dashboard; providers cannot be assigned jobs | Roles |
| Medium | No SEO: client-rendered SPA, one title for every page, property cards are click handlers not links (can't open in new tab, not crawlable) | Search, listings |
| Medium | Stats band shows "—" for all four numbers on Home and About | Home, About |
| Medium | No lease document or e-signing; a "lease" is just a record created on approval | Leases |
| Medium | Live location tracking during viewings needs explicit consent, time limits and clear stop controls | Viewings |
| Low | After AI search sets max rent R4,000, the Max Rent dropdown shows blank (value not in its option list); the Furnished parse is not shown as a chip | Search |
| Low | "1 properties found" grammar; \~5 s AI search with only a spinner | Search |
| Low | Tenant's in-dashboard search lacks the AI search box the public search has; two search UIs to maintain | `/tenant/search` |
| Low | Viewing booking takes any date and time; no landlord availability or double-booking check | Listing page |
| Low | Some tenant pages (Viewings, Payments) render blank for a few seconds before data loads, with no skeleton; the apply form shows a large empty area below the fold | Tenant dashboard |
| Low | No light mode, "Built with Hercules" badge on every page | Global |
| Low | Header has no notification bell or unread message count on public pages | Header |

## Rebuild recommendations and phasing

Rebuild in four phases. Ship the marketplace core first, because everything else hangs off listings and applications.

**Suggested stack:** Next.js (server-rendered listing and search pages for SEO) with Tailwind + shadcn/ui to match the current look. Backend: either a .NET or Node API on PostgreSQL (with PostGIS for map search), or keep a managed backend like Supabase. Object storage (S3 or R2) for photos and documents, with private buckets for tenant documents. Your own auth with email, Google and phone OTP. Transactional email/SMS/WhatsApp for notifications.

1. **Phase 1, marketplace core:** auth + onboarding roles, public pages, property CRUD with photos, admin approval queue, search with filters, listing page with map, saved properties.
2. **Phase 2, rental workflow:** viewings, applications with document upload, approve/reject, leases, messaging, notifications, settings and preferences.
3. **Phase 3, operations:** maintenance requests with a real Maintenance Provider role and job assignment, rent payment records, landlord analytics, admin analytics, reviews.
4. **Phase 4, differentiators:** AI listing generator and NL search, an SA payment gateway with reconciliation, lease PDF + e-signature, live viewing tracking (opt-in), agent-to-owner relationships.

**Before you start, get answers to these from the client:**

- [ ] Should Estate Agent, Property Manager and Maintenance Provider get their own features, or be merged into Landlord for v1?
- [ ] Is online rent collection in scope, and through which provider?
- [ ] Is live location tracking during viewings a must-have?
- [ ] Can the 7 existing listings and any user accounts be exported from Hercules/Convex, or do we start empty?
- [ ] Who writes the Privacy Policy and Terms, and what is the document retention period?
- [ ] Final domain and brand assets (logo files, colours), and is the dark-only theme intentional?
- [ ] Any monetisation: listing fees, featured listings, or commission?
- [ ] Is there an existing admin account? Who holds it, and how was it created? We need admin access to verify the Admin panel
