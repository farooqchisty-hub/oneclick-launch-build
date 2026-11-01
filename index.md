# One-Click Launch: build doc

Last updated 1 Nov 2026. Companion docs: [SaaS plan](SAAS_PLAN) (architecture and launch plan) and [Claude Design prompt](CLAUDE_DESIGN_PROMPT) (landing page brief).

---

## 1. Where things stand

| Area | State |
|---|---|
| Delivery engine (Cloudflare Worker `ocl-core`) | Live. Crawl to kit, plan, drip, live submissions, inbox, vault, Kernel stealth, Google lane, badge lane, proof, UTM links, recipes, sweeper, retries, pacing. Models on OpenRouter (gpt-6-luna, haiku-5.5 fallback). |
| Catalog | 796 directories researched, 545 enabled. Lanes: automated, stealth (bot checks), badge, Google sign-in. Niche directories carry fit tags. |
| Pilot 1: Ad Studio (ads.brevo.com) | 134+ accepted, Google lane running at 25/day. |
| Pilot 2: PushOwl (pushowl.com) | Live since 1 Nov, 400 planned. No Google identity yet. |
| Pilot 3: Brevo (brevo.com) | Live since 1 Nov, 400 planned. No Google identity yet. |
| Landing page (agent build, stand-in) | https://oneclick-launch.pages.dev/site/ with the real `POST /api/preview` |
| Customer + admin dashboards | https://oneclick-launch.pages.dev/app/ (admin at `#/admin`, key in sessionStorage for now) |
| SEO and traffic metrics of directories | Agent running (Ahrefs DR, refdomains, organic traffic; Similarweb visits; dofollow check). Output: `research/directory_metrics.csv` and `_summary.md`. |
| Payments, auth, onboarding | Not built. Designed below. |

## 2. Customer flow (designed)

### 2.1 Before paying (landing page)
1. Visitor enters their URL in the hero input.
2. **Preview** (live today): their name and logo, directories matched by type, 6 sample listings written from their site.
3. **"Your launch, in progress" teaser:** a blurred mock of their dashboard, using their real name and logo and real directory names:
   - a progress bar at, say, 37%;
   - rows like "Toolify: listed", "There's An AI For That: in review", "Futurepedia: submitting".
   - A lock overlay: "Start my launch".
   - This shows exactly what they're buying. The blur is honest: it's labelled "Preview of your dashboard".
4. The CTA scrolls to **Pricing**, and the plan button opens **Stripe Checkout** (one-time payment). The product URL rides along in the Checkout metadata; Stripe collects the email.

### 2.2 After paying
1. The Stripe webhook (`checkout.session.completed`) creates:
   - the **user** (by the email Stripe collected);
   - the **order** (plan, paid amount, timestamps, due dates: T+24h and T+48h);
   - the **product** (the URL from metadata).
   The profile job starts immediately; the preview cache makes it near-instant.
2. Stripe redirects to `/app/welcome?session=...`. The page shows "Payment received, your launch is starting" and the sign-in step:
   - **Continue with Google** (Google OAuth for our app; the email must match, or the order is linked to the Google account explicitly);
   - **Email me a sign-in link** (magic link sent by Brevo transactional; the email is prefilled from Stripe and the link is sent automatically on arrival).
3. **Onboarding (one screen, all optional except confirming):**
   - **Website:** prefilled, confirm.
   - **Where directory emails go:** the account email by default (a daily digest, or forward everything).
   - **Listing kit review:** name, taglines, descriptions, categories, logo and screenshots, with an "Edit" option. It auto-approves after 10 minutes so the launch never waits on them.
   - Nothing else. Accounts, inboxes, Google identity and verification are all ours.
4. **Dashboard:** progress, countdowns, live feed, listings with proof, inbox, badge block (optional), final report at 48h.

### 2.3 Accounts and data model to add
- `users` (id, email, google_sub, created_at)
- `sessions` (token hash, user_id, expires)
- `orders` (id, user_id, product_id, plan, amount, currency, stripe_session, status, paid_at, due_24h, due_48h)
- `products.user_id`
- Admin is behind Cloudflare Access (replaces the shared key).

## 3. Plans (proposal; final cuts wait for the metrics agent)

Every listing we do falls into one of these value buckets:
- **Reach:** count of submissions.
- **Authority:** DR of the directory.
- **Link value:** dofollow or not.
- **Audience:** traffic.
- **Fit:** category directories, such as AI-only, marketing or dev tools.
- **Effort we absorb:** Google sign-in, bot checks, badges.

| Plan | What's in it | Why someone pays more |
|---|---|---|
| **Launch** (one-time, base price) | Automated + stealth + badge-submit directories (about 300 to 350); listing kit; proof; UTM links; 48h report | Fastest path to "listed everywhere that's free" |
| **Pro** (one-time, about 2x) | Launch, plus the Google sign-in directories (about 180, on our Workspace or Cloud Identity identity); a **High-authority pack** (DR 40+ first, with priority retries); a **Dofollow pack** report; a **Category pack** matched to their product (AI-only, marketing, dev tools, ecommerce tools, regional); the badge footer block set up; a 30-day traffic report from the UTM links | More and better links: authority and dofollow are what SEO buyers pay for |
| **Always-on** (monthly) | Pro, plus monthly submission to new directories as the catalog grows (research agents add about 50 to 70 a month), re-submission when a listing drops, and multi-product (up to 3 products) | Catalog freshness becomes recurring revenue |

To decide after the metrics land: the exact DR threshold, how many are dofollow, combined traffic per tier, and prices. Unit cost per order is about $2 to $4 (Pro higher with Google: Kernel time).

## 4. Google identities (Cloud Identity Free on thereisnomoat.fyi)

Why: Google-only directories need a Google account per customer. Consumer Gmail can't be created by a product (phone checks, bulk-creation bans). Cloud Identity Free gives free, admin-created Google accounts on our domain (no Gmail; mail keeps flowing to our catch-all inbox). The default cap is 50 users, and more on request.

**Steps for Farooq** (I can't create the account or the API key for you):
1. Open https://workspace.google.com/gcpidentity/signup?sku=identitybasic (Cloud Identity Free).
2. Business name, "Just you", country, then your contact email (your Gmail).
3. "Does your business have a domain?" Yes: `thereisnomoat.fyi`.
4. Create the admin user, e.g. `admin@thereisnomoat.fyi`, and save the password in 1Password.
5. Domain verification: Google shows a TXT record (`google-site-verification=...`). Tell me and I'll add it in Cloudflare through your Chrome, or add it yourself. **Do not change MX records** if Google suggests it (no Gmail here; the catch-all inbox must stay).
6. Google Cloud console (https://console.cloud.google.com), signed in as that admin:
   - create a project `ocl-identity`;
   - enable **Admin SDK API**;
   - IAM, then Service accounts: create `ocl-provisioner`, then Keys, then add a JSON key, and save it as `~/.gcp_identity_key.json`.
7. Admin console (https://admin.google.com): Security, then Access and data control, then API controls, then **Domain-wide delegation**, then Add new.
   - Client ID: the service account's client ID.
   - Scope: `https://www.googleapis.com/auth/admin.directory.user`.
8. Tell me "identity ready". I'll create 3 test identities, sign each in from a Kernel browser, and run a few Google-only directories to see whether Google challenges brand-new accounts.

## 5. Decisions pending (yours)

1. **Product name and main domain** (working name "LaunchKit" is a placeholder).
2. **Landing page:** iterate in Claude Design with the prompt and send me the locked export.
3. **Plans and prices:** Launch, Pro, Always-on as above, or simpler. Base price placeholder is $49.
4. **Guarantee or refund wording,** e.g. "100+ submissions or we refund the difference".
5. **Stripe account:** which entity charges (personal, a new company, or Brevo).
6. **Google identities:** Cloud Identity setup (section 4). Fallback: Workspace at $7 to $8.40 per user per month.
7. **Directory emails to customers:** a daily digest (proposed default) or forward everything.
8. **Data retention:** identities kept (free), customer data deleted 90 days after delivery (proposed).
9. **Badge block for the pilot:** whether ads.brevo.com, pushowl.com and brevo.com add a "Featured on" footer block (unlocks the badge-gated directories).
10. **Legal:** terms and privacy text (lawyer, or a template and review).

## 6. Build order from here

1. Finish the pilots (3 products) and fix the failure clusters they show (top: forms that never advance).
2. Google identities (once section 4 is done), then give PushOwl and Brevo the Google lane.
3. Accounts: users, sessions, Google OAuth for our app, magic link via Brevo.
4. Orders and Stripe Checkout plus the webhook; the deadline scheduler keyed by order.
5. Onboarding screen and the blurred teaser on the landing page (on the Claude Design version).
6. Admin: real endpoints for the mocked panels (ops, directories, alerts, cost), Cloudflare Access, alert delivery (Slack or email).
7. Plans: tier logic in the planner (Launch vs Pro lanes and packs) once the metrics land.
8. Legal pages, brand and domains, soft launch with 10 paying customers.

## 7. Notes and explanations

- **Admin key:** the dashboard uses the shared admin key until Cloudflare Access is set up. Rotate it after Access is live.
- **Google lane on Ad Studio:** runs in one signed-in Kernel browser, paced (1 per 12 minutes, 25 a day, a 6-hour pause after any Google challenge). The browser lives 72 hours idle, so if the lane stops for 3 days you'll need to sign in again.
- **Known engine debt:** forms that never advance (largest cluster), status-text noise from other forms on the page, and Kernel's visible-captcha solver (skipped by your decision).
- **Keys used during the build that should be rotated before launch:** the Kernel key (pasted in chat) and the admin key.
