# One-Click Launch: SaaS build plan (v1, 1 Nov 2026)

Goal: a customer pastes a URL, pays once, and within 48 hours has every listing we can get, with proof, tracked links and zero manual work. Automated lanes inside 24 hours.

---

## 1. Decisions

### 1.1 Google sign-in: our own Workspace identities, not consumer Gmail

- **No consumer Gmail accounts created by the product.** Google requires phone verification for new consumer accounts and bans accounts created in bulk or by automation. They would be disabled, and the listings under them would be orphaned.
- **Instead, Google Workspace on a dedicated domain we own** (e.g. `listings-id.com`, never the customer's or Brevo's domain).
  - At purchase, the backend creates one Workspace user per order through the Admin SDK Directory API, e.g. `adstudio-7k2@listings-id.com`. This is a normal, admin-created account: no phone check and no customer action.
  - The password is generated and sealed in our vault (AES-GCM, as for directory accounts). No 2-step verification on these users.
  - A Workspace routing rule forwards all mail to our inbox Worker. The customer gets a daily digest plus instant forwards of anything that needs them.
  - The user is suspended 30 days after delivery, which stops billing. It stays recoverable if a directory later needs re-verification.
- **Cost:** the Workspace Flexible plan is billed per user and prorated. A user alive for the delivery window plus suspension is roughly $0.50 to $1 per order. Confirm on the first invoice.
- **Signing in:** the engine signs the Workspace user in once, in that order's long-lived Kernel browser. Google lane directories then run as tabs in it (already built: attach with `persistent: true`, account chooser and consent clicks).
- **Risks to test first:**
  - Google may still challenge a brand-new Workspace user signing in from a cloud IP. Mitigations:
    - pin a Kernel ISP proxy per identity, for a stable IP;
    - warm each identity with a normal Google page visit before the first OAuth;
    - admin console "turn off login challenge" as the fallback.
  - Test with 3 identities before relying on it.
- **Pacing:** the risk now sits on our accounts, not the customer's, so the pace rises.
  - Speed: one Google sign-in every 10 minutes per identity, at most 150 a day.
  - Challenges: on any challenge, pause that identity for 2 hours, then retry. Two challenges in a day suspend it from the lane and alert the admin.
  - Fit: 131 Google directories at 1 every 10 minutes is about 22 hours, which fits the 48-hour window.
- **The customer's own Google remains an option**, as an "Use my Google account" toggle for customers who want listings under their own identity.

### 1.2 Delivery promise

| Lane | What it is | Target |
|---|---|---|
| A: automated | open forms, email signup, magic links | 95% of attempts finished in 12h, all in 24h |
| B: stealth | hidden bot checks via Kernel | 24h |
| C: badge | submitted on the free-with-badge option | 24h (listing holds once the customer adds the optional footer block) |
| G: Google | Workspace identity, paced | 48h |
| Review-side | directory reviews (days to months) | outside our clock: we report "submitted, in review" with proof |

The promise is **submission with proof inside 48 hours**. Approval and permanence are the directory's call; the terms and the landing page say so.

### 1.3 What "done" means per directory (final states the customer sees)

- **Listed:** a live listing URL was found.
- **Submitted, in review:** a confirmation on the page or by email, plus a proof screenshot.
- **Submitted, badge pending:** listed on the free-with-badge path. Optional: add the footer block.
- **Couldn't list:** the directory refused or closed. Shown with the reason and not counted in the promise number.
- There is no "needs you" in the default flow. The engine retries until the deadline, then it becomes "couldn't list".

---

## 2. Delivery engine changes (backend)

What exists and is proven: crawl to kit, plan builder with fit tags, drip queue, live engine (sign-up, mail verification in-run, codes, magic links, uploads, selects, math checks, consent, recipes replayed without a model, step trails), inbox Worker, vault, Kernel stealth plus attach, Google lane, badge block, proof screenshots, tracked UTM links, OpenRouter models.

To build:

1. **Order scheduler with deadlines** (replaces the daily-cap drip).
   - Each order gets a delivery plan: lane, priority, `due_at` (A/B/C at T+24h, G at T+48h).
   - The scheduler fills worker slots by earliest deadline, round-robin across orders, so one big order never starves others.
2. **Retry policy per attempt.**
   - Attempt 1: Browser Run.
   - Attempt 2 (+30 min): fresh session and fresh model mapping.
   - Attempt 3 (+2h): Kernel stealth browser.
   - Attempt 4 (+8h): Kernel with a recipe from another order on the same directory, if one exists.
   - Mail waits: awaiting_email re-checks at +15 min, +1h and +4h.
   - Stop early on hard outcomes: paywall, closed, declined.
3. **Stuck-run sweeper.** A listing still "running" after 15 minutes (killed by a deploy or a crash) is requeued automatically. This removes the deploy-kills-runs problem.
4. **Directory health** (catalog-wide, learned from every order).
   - Track rolling success over the last 20 attempts per directory.
   - Below 20%: auto-pause the directory and alert. A weekly dry-run health check re-enables it.
   - Recipe versions are kept per directory; when a recipe fails twice, the model re-maps and the new recipe is stored (already the pattern).
5. **Per-directory rate limits.**
   - At most N submissions per directory per hour across all customers (default 6), so directories don't see a flood from one IP range and one mail domain.
   - Inbox domains are rotated across a small pool (3 to 5 domains); each order sticks to one.
6. **Capacity.** At about 350 directories per order and about 3 minutes per run, one order is about 17 browser-hours.
   - Queue `max_concurrency` goes from 10 to 50 or more.
   - Browser Run limits: check the paid limits and request a raise.
   - Kernel: Start-Up plan (150 concurrent) once Google and captcha lanes run per order.
   - Target: 20 orders a day comfortably inside the window.
7. **Multi-product correctness.** Every query is keyed by order or product (mostly true already). Remove the single-pilot assumptions (product id in scripts, DAILY_LIVE per product).
8. **Intake checks.**
   - A model screens the URL first: refuse adult, scam, malware, gambling and spam sites, which would get our identities banned everywhere.
   - One product per order; the site must load and have a real product.
9. **Cost meter per order.** Record LLM tokens and cost (OpenRouter returns usage.cost), Kernel GB-seconds, Browser Run minutes and Workspace days. Shown in the admin dashboard; alert if an order goes over 3x the average.

---

## 3. Customer product (UI/UX)

### 3.1 Landing page
- Hero: a URL input with a "See my directories" button, which runs an instant preview (no signup):
  - the crawled name and logo;
  - the number of matched directories by lane;
  - 6 sample listing cards written from their site.
- Proof section: live counters, sample proof screenshots, and a timeline: "most listings in 24 hours, the rest within 48".
- How it works (3 steps), the honest caveats (directories review on their own schedule; some may require a badge to keep the listing), FAQ, pricing and checkout.
- Brand and name are still parked: needs a name, a domain and a logo.

### 3.2 Checkout to onboarding
- Stripe Checkout, one-time. Collect the email and the product URL (prefilled from the preview).
- After payment, a magic-link sign-in to the dashboard.
- A 10-minute review window on the generated listing kit (name, taglines, descriptions, categories, logo, screenshots). It auto-approves if untouched, so there's still no required work.
- Optional toggles: forward all directory emails (default: daily digest), use my own Google account (default: ours), show the "Featured on" block.

### 3.3 Customer dashboard
- A progress header: X of Y done, a per-lane bar, and a 24h and 48h countdown.
- A live feed ("Listed on Toolify 2 min ago").
- A listings table with directory, status, listing link, proof screenshot, tracked link and date, plus filters and CSV export.
- An inbox tab with directory mail (verification already handled, replies needing them highlighted).
- The badge block (optional) with a copy button and a live check.
- The final report at 48h, emailed as PDF and CSV.

### 3.4 Emails (sent through Brevo)
Order confirmed, the 24h report, the 48h final report, digests, and a delivery-at-risk notice (sent only if the admin hasn't fixed it).

---

## 4. Admin dashboard (for Farooq)

- **Orders:** status, SLA health (on track, at risk, late), per-lane progress, cost so far, and a customer view (impersonate read-only).
- **Live ops:** queue depth, runs per minute, success rate in the last hour, Kernel and Browser Run concurrency in use, and Google identity states (signed in, paused, challenged).
- **Directory health:** success rate, last success, common failure reason, recipe version, paused or enabled, and buttons for dry run, re-enable and edit submit URL.
- **Run explorer:** any run's step trail, screenshots and model calls, with retry, force-Kernel and mark-listed buttons.
- **Failures clustered by cause** (same error text across directories means an engine bug, not a directory problem).
- **Alerts** (Slack and email):
  - an order's SLA is at risk;
  - a directory's success rate collapses;
  - the queue is stuck;
  - the error rate spikes;
  - the OpenRouter, Kernel or Workspace balance or limit is low;
  - a Google identity is challenged twice;
  - an inbox domain is bouncing.
- Access is behind Cloudflare Access, with no shared admin key in URLs.

---

## 5. Everything else needed for launch

- **Legal:**
  - Terms: we create accounts and submit on your behalf; approval and permanence are the directory's call; no guarantee of rankings or traffic.
  - Privacy policy, data retention (identities suspended at 30 days, data deleted at 90), and a DPA for EU customers.
  - Consent wording at checkout.
- **Abuse and reputation:** URL screening, refusing re-submissions of the same domain within 30 days, and per-directory caps. Protecting directory relationships is the moat.
- **Security:**
  - Vaulted credentials and per-order identities.
  - Secrets only as Worker secrets; rotate the keys used during the build (Kernel key was pasted in chat).
  - Admin behind Access, and rate limits on public endpoints (the preview is costly: cap it per IP and cache by domain).
- **Reliability:** the sweeper, idempotent jobs, deploys gated on the queue draining, the D1 backup schedule, R2 lifecycle for screenshots (keep 180 days).
- **Observability:** Workers Logs or Logpush, structured run events, error capture, and a daily ops summary email.
- **Support:** a help widget or email, a "report a listing problem" button per row, and a status page.
- **Refund policy:** one-time product; offer a partial credit when fewer than N listings are submitted. Product decision; the default proposal is a guarantee number (e.g. "100+ submissions or we refund the difference").
- **Analytics:** PostHog on the landing page and dashboard (preview to checkout funnel); attribution of the customer's own UTM traffic per directory on the 7 and 30-day report, which becomes the upsell hook ("your top 10 traffic directories").
- **Pricing and economics:** cost per order is about $2 to $4 (LLM about $0.30, Kernel about $1 to $2, Workspace about $0.60, browser minutes about $0.50). Validate on pilots before setting the price.
- **Catalog upkeep:** a weekly dry-run sweep, a monthly research agent pass for new directories, and auto-pause on failure. The 791-directory catalog is the asset; it needs an owner process.
- **Brand and domains:** product name, main domain, 3 to 5 inbox domains, a Workspace domain, and SPF/DKIM/DMARC on all of them.

---

## 6. Battle-testing before launch

Ad Studio is one product. Before selling:
1. Run 5 pilot products across types: a non-AI SaaS, a dev tool, an open-source project, an ecommerce app and a consumer web app. Ideally use real friendly founders.
2. Measure per lane: submitted, listed after 7 days, and time to done. Target at least 100 submissions per product.
3. Run 3 orders at once to prove fairness, rate limits and capacity.
4. Run a regression suite: replay the 130 proven recipes weekly against a test product and alert on breakage.
5. Fix every engine-level failure cluster found (the current biggest: forms that never advance, at 181; read their trails in bulk and fix generic causes).

---

## 7. Build order

1. **Engine hardening:** the sweeper, the deadline scheduler, the retry policy, directory health and rate limits, the cost meter, intake screening.
2. **Google Workspace identities:** domain plus the Admin SDK, a test with 3 identities, then wire into lane G.
3. **Multi-product pilot:** 5 products in parallel; fix failure clusters.
4. **Admin dashboard** (needed to run pilots well), then alerts.
5. **Customer dashboard plus magic-link auth plus emails.**
6. **Landing page with live preview.**
7. **Stripe Checkout** plus order creation webhook.
8. **Legal pages, brand and domains**, then soft launch to 10 paying customers with manual watch, then open.
