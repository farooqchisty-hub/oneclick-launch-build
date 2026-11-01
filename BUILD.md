# One-Click Launch: build doc

Last updated 2 Nov 2026. Companion to [SAAS_PLAN.md](SAAS_PLAN.md) (architecture and launch plan) and [CLAUDE_DESIGN_PROMPT.md](CLAUDE_DESIGN_PROMPT.md) (landing page brief).

---

## 1. Where things stand

| Area | State |
|---|---|
| Delivery engine (Cloudflare Worker `ocl-core`) | Live. Crawl to kit, plan, drip, live submissions, inbox, vault, Kernel stealth, Google lane, badge lane, proof, UTM links, recipes, sweeper, retries, pacing. Models on OpenRouter (gpt-6-luna, haiku-5.5 fallback). |
| Catalog | 839 directories, about 760 enabled. Each product is planned on 754. |
| Pilot 1: Ad Studio (ads.brevo.com) | 161 accepted. 126 Google directories queued (lane paused by Google phone prompts). |
| Pilot 2: PushOwl (pushowl.com) | 139 accepted. Re-planned from 319 to 754 directories on 1 Nov. |
| Pilot 3: Brevo (brevo.com) | 142 accepted. Re-planned from 319 to 754 directories on 1 Nov. |
| Admin dashboard | Live data: ops, directory health, cost per product, run explorer, live feed, inbox, alerts, admin actions. https://oneclick-launch.pages.dev/app/#/admin |
| Customer side | Built and deployed: accounts (email link, Google), Stripe checkout and webhook, orders that launch themselves with 24h and 48h deadlines, welcome, one-screen onboarding, customer dashboard on real data. Waiting on Stripe, Brevo and Google OAuth keys. |
| In progress | Pro plan gates the Google lane, daily mail digest or forward-all, refunds, renewals, delivery marking, CSV report. |
| Landing page | Agent stand-in at https://oneclick-launch.pages.dev/site/ (Claude Design version pending). |

### Engine changes on 1 Nov
- A directory is switched off only after 3 different products fail on it. Before, one miss switched it off for everyone: 268 directories had been dropped that way (40 had worked for another product). All are back on.
- Required fields are never left empty: missing links get the website URL, text boxes get the closest kit copy, categories get the closest option, any empty email box gets the product's directory inbox.
- Inbox: the agent sends a magic link before waiting for it, enters code-only verification mails, and accepts Firebase sign-in links.
- 43 qualified directories from the research passes imported; 5 Google-only directories moved to the Google lane; 12 "complex" directories re-enabled.
- Google lane browser uses one static ISP exit IP from the next sign-in.

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

## 3a. Network metrics (measured 1 Nov 2026)

539 directories measured: Ahrefs DR, referring domains and organic traffic (from Ahrefs' public checkers, via Apify), plus Similarweb monthly visits and a dofollow check of one listing page per directory. Cost: about $8.

| | Directories | Monthly visits | Organic traffic | Median DR |
|---|---|---|---|---|
| Whole catalog | 539 | 24.3M | 7.9M | 32 |
| Accepted the pilot (Ad Studio) | 137 | 7.5M | 2.4M | 21 |

**DR spread:**

| DR band | Directories |
|---|---|
| 70+ | 44 |
| 50 to 69 | 81 |
| 30 to 49 | 166 |
| 10 to 29 | 133 |
| 0 to 9 | 105 |

**Dofollow:** 198 confirmed, 167 nofollow, 174 unknown (bot walls or links rendered by script).

**Caveats:**
- The top 5 sites carry 43% of visits.
- 114 of the 211 DR 40+ sites get under 100 organic visits a month, which usually means link-built authority.
- Dofollow is one sampled page per site: spot-check before selling a dofollow guarantee.

**Packs the data supports:**

| Pack | Directories | Combined visits |
|---|---|---|
| Proven authority (DR 40+ and 5K+ visits) | 122 | 20.6M |
| DR 70+ | 44 | 14.3M |
| Dofollow | 198 | 5.3M |
| Dofollow and DR 40+ | 61 | 3.7M |
| 100K+ visits | 41 | 19.8M |
| AI directories | 115 | 2.1M |

**Marketing lines we can honestly claim (rounded down):**
- 500+ launch directories.
- Over 24 million visits a month combined (Similarweb estimates).
- Over 200 directories at Ahrefs DR 40+, and 40+ at DR 70+.
- Over 140 directories that each get 10,000+ visits a month.
- Nearly 200 directories give dofollow links.
- Our pilot product was submitted to 130+ directories with a combined 7 million monthly visits.

The engine now runs high-DR directories first in every plan (DR stored as directory authority).

**Proposed plan cut-offs:**
- **Launch:** the whole catalog, with a guaranteed floor of successful submissions.
- **Pro:** adds the Google sign-in directories, priority and extra retries on the Proven authority pack (122) and the Dofollow pack (198), and the 30-day traffic report.
- **Always-on:** monthly.

## 4. Google lane (Google accounts with a software passkey)

About 200 directories only accept "Sign in with Google". What we learned on 1 and 2 Nov:
- **Brand-new accounts are blocked.** Cloud Identity users on thereisnomoat.fyi got "Couldn't sign you in" at the first sign-in, even on a real Mac with a phone code. Only older Google accounts work.
- **Every new directory triggers "Verify it's you".** The only choices offered are a phone tap or a passkey (no authenticator codes).
- **The fix:** each Google account holds a software passkey (a WebAuthn credential the engine keeps sealed). When Google asks, the engine loads it on Google's tab and clicks "Use your passkey". No person, no phone.

**Where accounts come from**
- **Pool:** older Gmails we own (agentsyshop@gmail.com today). One pool account serves up to 5 products; products take one automatically when they get the Google lane.
- **Customer's own (optional):** at checkout and on the dashboard, "Get about 200 more listings: connect a Google account". Honest copy: what the passkey is for, that every sign-in is listed, that we remove it and sign out when the launch finishes, and to use a work or secondary account.

**Connecting an account (about 2 minutes of a person)**
1. Admin, Google accounts, enter the Gmail, **Connect a Gmail** (or the customer's Connect Google button). A cloud browser window opens on Google's sign-in page; sign in with the password.
2. **Start passkey step**, then click **Create a passkey** in the window (if Google says it can't, **Use another device**). The engine attaches the software passkey device and saves the credential.
3. **Approve** the new passkey on a phone or laptop already signed in to that account (Google Account, Passkeys, Approve). Otherwise Google holds new passkeys for a security delay.
4. Status turns **ready** by itself; the lane starts.

**How it runs**
- One long-lived Kernel browser per account (static ISP exit IP). Pacing and pauses are per account: one Google sign-in every 5 minutes across all its products, at most 80 a day, a 6-hour pause if Google still stops a sign-in.
- Customer accounts are removed when the order is delivered: the engine deletes our passkey on Google's passkeys page, signs out and deletes the browser. A failure raises a high alert with manual steps.

**Routes**
- Admin: `GET/POST /api/admin/google-accounts`, `GET /api/admin/google-accounts/:id`, `POST .../:id/passkey`, `POST .../:id/remove`, `POST .../:id/unpause`, `POST /api/admin/google-accounts/assign/:productId`.
- Customer: `GET /api/orders/:id/google` (status and sign-in log), `POST /api/orders/:id/google/connect`, `POST /api/orders/:id/google/passkey`.

## 5. Decisions pending (yours)

1. **Product name and main domain** (working name "LaunchKit" is a placeholder).
2. **Landing page:** iterate in Claude Design with the prompt and send me the locked export.
3. **Plans and prices:** Launch, Pro, Always-on as above, or simpler. Base price placeholder is $49.
4. **Guarantee or refund wording,** e.g. "100+ submissions or we refund the difference".
5. **Stripe account:** which entity charges (personal, a new company, or Brevo).
6. **Google accounts:** more older Gmails for the pool (section 4). Cloud Identity accounts are blocked by Google as too new.
7. **Directory emails to customers:** a daily digest (proposed default) or forward everything.
8. **Data retention:** identities kept (free), customer data deleted 90 days after delivery (proposed).
9. **Badge block for the pilot:** whether ads.brevo.com, pushowl.com and brevo.com add a "Featured on" footer block (unlocks the badge-gated directories).
10. **Legal:** terms and privacy text (lawyer, or a template and review).

## 6. Build order from here

1. Finish the pilots (3 products) and fix the failure clusters they show (top: forms that never advance).
2. Google lane: done on all three pilots through the pool account (section 4); add pool Gmails as they come.
3. Accounts: users, sessions, Google OAuth for our app, magic link via Brevo.
4. Orders and Stripe Checkout plus the webhook; the deadline scheduler keyed by order.
5. Onboarding screen and the blurred teaser on the landing page (on the Claude Design version).
6. Admin: real endpoints for the mocked panels (ops, directories, alerts, cost), Cloudflare Access, alert delivery (Slack or email).
7. Plans: tier logic in the planner (Launch vs Pro lanes and packs) once the metrics land.
8. Legal pages, brand and domains, soft launch with 10 paying customers.

## 7. Notes and explanations

- **Admin key:** the dashboard uses the shared admin key until Cloudflare Access is set up. Rotate it after Access is live.
- **Google lane:** each Google account lives in one signed-in Kernel browser (section 4). The browser lives 72 hours idle, so an account whose lane stops for 3 days needs signing in again.
- **Known engine debt:** forms that never advance (largest cluster), status-text noise from other forms on the page, and image-text captchas Kernel's solver does not take (reCAPTCHA, hCaptcha and Cloudflare checks are solved).
- **Keys used during the build that should be rotated before launch:** the Kernel key (pasted in chat) and the admin key.
