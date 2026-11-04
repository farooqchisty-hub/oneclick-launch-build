# One-Click Launch: engine playbook

The rulebook for the delivery engine (Cloudflare Worker `ocl-core`): what it must always do, the trap that taught each rule, where the rule lives in code, and whether it is enforced. Written 4 Nov 2026 from the code in `app/src`, the deploy log, the project history and the live D1 database.

How to read a rule: **Rule** (one line), **Trap** (the symptom that taught it), **Code** (file and function; `runLive (x)` names the part of the long `runLive` function), **Status**:

- **Codified**: enforced in code today.
- **Documented**: a practice people follow; nothing in code enforces it.
- **Gap**: not handled; a risk for paying customers.

Lanes: **A** open forms (Cloudflare Browser Run), **B** captcha sites (Kernel stealth browser with its solver), **C** badge directories, **G** Google sign-in only (Kernel browser holding a Google account).

---

## Launch readiness

### Snapshot (D1, 4 Nov 2026)

| Area | Today |
|---|---|
| Catalog | 1,033 directories, 878 enabled (A 403, B 121, C 90, G 264). 155 disabled: 52 by a paywall verdict, 100 by the miss rule (81 not_submitted, 11 no_submit_button, 8 complex), 3 by hand. |
| Pilot results | 1,008 accepted listings across 3 pilots (A 713, B 163, C 106, G 26). 184 verified live, 785 not seen yet, 46 on directories that block the check. |
| Offer verdicts (D1 `directory_offer`, `src/offer.ts`, 4 Nov 23:02 IST) | All 1,033 directories: **426 proven** (108 seen live, 281 accepted, 37 sent unconfirmed), **81 badge**, **478 trying** (188 our form misses, 119 untried, 103 no form found yet, 36 captcha, 13 in progress, 19 other), **48 excluded** (35 paid only, 7 submit page not found, 3 offline, 3 not for products). Of the 968 enabled: 419 proven, 76 badge, 461 trying, 12 excluded. A paid Pro plan is planned on 949 of the 960 eligible enabled directories. See [Directory verdicts](#directory-verdicts). |
| Throughput | 734 live runs in the last 24 h, 2.2 min average, 333 of them in Kernel. 884 listings have been run more than 3 times. |
| Google lane | 142 of 144 runs signed in after the passkey fix. The 2 failures were the account-chooser trap (GOO-6); fix deployed 4 Nov 13:27 UTC, lane pause cleared. |
| Human checks | 9 tabs parked, 3 check browsers alive (the cap), oldest 18 h, holding 3 of the 7 Kernel stealth slots. |
| Worker secrets | Set: ADMIN_TOKEN, VIEW_TOKEN, OPENROUTER_API_KEY, REPLICATE_API_TOKEN, KERNEL_API_KEY, VAULT_KEY, GOOGLE_ADMIN_OAUTH. **Not set**: STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET, BREVO_API_KEY, MAIL_FROM, GOOGLE_OAUTH_CLIENT_ID/SECRET, ALERT_WEBHOOK, TREG_TOKEN. |

### What is codified

- **Delivery loop**: plan, drip, per-directory pacing, sweeper, stealth retries, mail-driven reruns, delivery at 48 h, refund check. Nothing is silently lost: a killed run is requeued within 10 minutes.
- **Accounts and inbox**: one sealed identity per product and directory, catch-all inbox, verify links and codes used inside the same browser session, password reset and +alias recovery.
- **Form engine**: about 60 field, popup, picker and submit rules (sections 5 and 6), proven recipes replayed without the model.
- **Success detection**: only text that appeared after the click counts; receipts, toasts, wizard steps, Google screens and sign-in pages are never a submission.
- **Google lane**: pooled Gmail accounts in long-lived Kernel browsers, software passkey for every "Verify it's you", per-account pacing and pause, chooser locked to the pool account.
- **Liveness**: "live" means seen on the directory, from a 9-step ladder with a strict judge.
- **Customer view**: one status mapping, lane deadlines, no engine internals shown.
- **Captcha stance**: Kernel's built-in solver as configured and nothing beyond it (KER-1).
- **Directory verdicts**: every directory has a stored verdict (proven, badge, trying, excluded) with its reason and evidence, recomputed daily from every product's listings, runs and liveness checks; paid plans are ordered by it and leave out only excluded ones ([Directory verdicts](#directory-verdicts)).

### Gaps, ranked by risk for paying customers

| # | Gap | Evidence | Risk | Fix |
|---|---|---|---|---|
| 1 | ~~Plans include directories already proven unofferable.~~ **Fixed 4 Nov** ([Directory verdicts](#directory-verdicts)). | Paid plans now leave out the 12 enabled directories excluded as paid only, dead or not for products, and run proven first (419), then badge (76), then trying (461). | What is left: trying directories still produce "Couldn't list" rows (by the owner's rule they stay in every plan, including 3 mail-only, 2 phone-code and 1 bot-walled ones), and proven directories a runtime paywall reading switched off (7) stay out of plans until a person switches them on (gap 2). | Keep fixing trying reasons (form rounds); a person reviews the switched-off proven list in the admin Directories view. |
| 2 | **Auto-disable on our own form misses.** `runLive (miss rule)` switches a directory off catalog-wide after 3 products miss (not_submitted, no_submit_button, complex, badge_required). | 100 directories off today. Paying customers are products, so the rule fires faster as orders arrive. A disabled directory is never run again, so its built-in re-enable (a later success) can never fire. Customers without a phone produce `complex` on phone-required forms and count toward the 3. No alert when it fires. Runtime also moves a directory to lane G on one run's verdict (CAT-12). | The catalog shrinks without anyone noticing; proven directories vanish from plans. | Being fixed by the form agent. Target: runtime code never writes `enabled`; misses move a directory to verdict `trying` and raise an alert. |
| 3 | **No regression check after deploys.** | 21 deploys on 3 Nov alone. The failure-cluster alert needs the same note on 5 directories within 24 h. | One bad detector change can mark hundreds of listings failed (or falsely accepted) before anyone sees it. | `scripts/canary.sh` (below) as the gate after every deploy; then a cron detector that compares each lane's acceptance rate in the hour after a deploy with the previous 24 h. |
| 4 | **Human-check queue holds browser slots while nobody ticks.** | 3 check browsers (the `CHECK_BROWSERS` cap) alive 18 h with 9 parked tabs. They count against `KERNEL_STEALTH_SLOTS` (7), so captcha runs get 4. A parked tab refills after 24 h, a browser is recycled after 72 h, and the cycle repeats. `needs_check` shows "In progress" forever. No alert, no owner. | Lane B slows for every customer; parked listings never resolve. | Alert when a tab is parked over 2 h; name an owner; end a check after 12 h unticked (status `check_expired`, shown as "Couldn't list: needs a human check"); delete a check browser nobody opened in 6 h. |
| 5 | **Nobody is paged.** | `ALERT_WEBHOOK` is not set, so high alerts live only on the admin dashboard. No detector for auto-disables, parked checks, a failed profile on a paid order, or a drop after a deploy. | Problems surface only when someone opens the dashboard. | Set `ALERT_WEBHOOK`; add the four detectors to `alerts.ts`. |
| 6 | **Opening the form follows OAuth buttons and "Free ..." product cards** (found by the canary, 4 Nov) | The click ranking in `collectOnce` treats any control that starts with "continue" or "free" as a way to the form. ainave.com: it pressed "Continue with Google" and landed on accounts.google.com. resource.fyi: it opened a product card named "Free App Logo". Live runs open forms the same way. | A lane A directory can be read as Google-only and moved to lane G for every customer (runtime has moved 3 directories; 2 had been accepted by email); an OAuth start from Browser Run breaks ACC-13. | Leave OAuth button texts and product cards out of the ranking; never move a directory's lane on one run (fold into the offer verdict). |
| 7 | Google lane capacity versus the 48 h lane deadline | One pool account serves all 3 pilots; pool max 5 products per account; 80 sign-ins a day per account; 264 enabled lane G directories per product. | A Pro customer's lane G alone needs 3+ days on one account; a lock on that one Gmail pauses every product on it. | Lane G is ordered by verdict since 4 Nov (22 proven and 39 badge run first, which fits in a day; 202 trying follow), never gated. Add pool accounts before selling Pro. |
| 8 | Customer phone | Only the pilots have a phone (migration 0008, one Indian number). The dial-code picker is hard-coded to India (+91) in `fillEmptyRequired` and `LIVE_SYSTEM`. | Phone-required forms end `complex` for customers (and feed gap 2); a non-Indian phone would be mis-filled. | Collect an optional phone at onboarding with its country; pick the dial code from it. |
| 9 | Status sets disagree | The accepted set is defined in 8 places. `refundCheck` and `directoryHealth` leave out `submitted_unconfirmed`, which the customer sees as "Submitted, in review". | A refund alert can fire while the customer's dashboard shows enough listings. | One shared status module imported by every file and `status.js`. |
| 10 | Recipes saved before acceptance | Recipes are saved on `awaiting_email` too: 468 of 1,112 live recipes; 55 directories (40 enabled) have a "proven" recipe and no accepted listing anywhere. | Replay repeats an unproven flow without the model. | Save on `submitted` only, or mark awaiting_email recipes unproven until the mail run accepts. |
| 11 | Crawl has no fallback | `runProfile` fetches over plain HTTP only. A script-only or bot-walled site ends `error`, and `launchPaid` never runs. | A paid order sits idle until the 24 h SLA alert. | Render in Browser Run when the fetch fails or the text is thin; alert on a failed profile for a paid order. |
| 12 | Password length | `newPassword` makes 20 characters; startup.info accepts at most 15. | Sign-up fails on such sites. | Read `maxlength` and the site's complaint; regenerate within it. |
| 13 | Plan limit | `launchPaid` and replans pass a limit of 1,000; 878 enabled today. | Past 1,000 enabled, plans truncate silently (the old 150 cap ran the drip dry at 89). | Plan the whole eligible set, or alert when truncated. |
| 14 | Dry-run browser launch outside `try` | `runDry` calls `launch()` before its `try`, so a launch failure leaves the run row `running` forever (1 such row today). | Admin and canary only. | Move the launch inside `try`. |

**Configuration blockers before taking money** (not engine work): Stripe keys and webhook secret, Brevo mail key and sender (customer sign-in links and mail), Google OAuth client, plan prices in `plans.ts` (marked NOT CONFIRMED), `ALERT_WEBHOOK`.

### Offer verdict (built 4 Nov)

The proposal that stood here is built: see [Directory verdicts](#directory-verdicts). Where the build differs from the proposal, the owner's 4 Nov rules decided: captcha directories (including a human check Kernel did not clear), phone-code, no-form and declined directories are `trying`, never `excluded`; paid plans get `trying` too (ordered after proven and badge), not only proven and badge; the verdict lives in its own table (`directory_offer`) and is recomputed daily, not hourly. Not built yet: alerts on a move out of proven, and `enabled` as a person's switch only (runtime still switches a directory off on one paywall reading, CAT-6, and on 3 products agreeing no_form, not_found or declined).

### Canary

`app/scripts/canary.sh` dry-runs 25 directories with proven recipes (8 lane A, 6 B, 5 C, 6 G) through `POST /api/admin/directories/:id/dryrun`, polls the `runs` rows in D1 (SELECT only) and prints pass or fail against the expected `page_state`.

- **Safe**: dry runs block every non-GET request and cancel every form submit (`runDry`), so nothing is submitted. Each entry uses a pilot that already has a listing on that directory, because a dry run points that product's `plan_items.last_run` at itself.
- **Expected state**: the first step of the directory's proven recipe, widened where a fresh dry browser legitimately reads differently: lane B may read `captcha_gate` (Browser Run has no solver), lane C may read `badge_required` (the dry prompt has no badge override), lane G may read any sign-in, sign-up or "Google only" verdict (a dry run cannot sign in with Google), and directories that need an account may read their sign-in or sign-up screen.
- **Fail** means: the page now reads blocked, not_found, no_form, paywall or other; the run errored or timed out; or a `form_ready` page had nothing filled (an engine regression in reading or filling).
- **Run**: `scripts/canary.sh` (all lanes) or `scripts/canary.sh A B`. Exit code 1 on any failure. `CANARY_CONCURRENCY` (default 3, Browser Run is shared with live runs), `CANARY_TIMEOUT` (default 420 s).
- **Limits**: it does not exercise submit, success detection, mail or Google sign-in. Those are covered by the post-deploy acceptance-rate detector proposed in gap 3.

**Results, 4 Nov 2026.** First run of all 25 (13:20 to 13:45 UTC; the 13:27 deploy landed during it and every dry run still finished): **17 of 25 passed**. The 8 failures:

| Directories | What happened | Action |
|---|---|---|
| aijet.cc, automationtools.directory, genify.dev, hubfortools.com (C) | Read `login_required` in a fresh browser. Their recipes start on the form because they were recorded signed in. | Canary mistake: expected states widened. Re-run: 5 of 5 lane C passed. |
| bluebook-directory.com, expansiondirectory.com (B) | Read `blocked` and an empty page: a request-verification interstitial on the submit.php network. ecodir.net from the same network passed, then read `blocked` 40 minutes later. | Not canary material in Browser Run: all three replaced (nextbigproduct.com, verifiedtools.info, aikendra.com). Re-run: 6 of 6 lane B passed. |
| ainave.com (A) | Opening the form pressed "Continue with Google" (ranked as "continue") and landed on accounts.google.com; the proven flow is an email sign-in. | **Real finding** (gap 6, FORM-22). Left failing until fixed. |
| resource.fyi (G) | The submit URL is the home page, and opening the form followed a product card named "Free App Logo". | **Real finding** (gap 6, FORM-22). Left failing until fixed. |

Current list: 23 entries pass; the 2 real findings fail until FORM-22 is fixed. A dry run of an account directory ends at its sign-in screen, so it proves the site answers and reads correctly, not that the sign-in works.

### Deploy procedure

1. Check nothing is mid-run: `SELECT COUNT(*) FROM listings WHERE status = 'running'`. A deploy kills in-flight runs (OPS-2).
2. Deploy, then append one line to the deploy log (what changed, which files).
3. Run `scripts/canary.sh`. Any failure: read that run's detail before the next deploy.
4. Watch the admin Alerts view for an hour (until `ALERT_WEBHOOK` is set).

---

## Directory verdicts

Every directory has one stored verdict that decides what a paying customer is planned on, in what order, what the offer page lists and what the admin is told. Table `directory_offer` (migration `0011_offer_verdict.sql`), code `src/offer.ts` (`judge` decides one directory, `computeOffers` writes them all), plan policy `src/plans.ts` (`PAID_OFFERS`, `offersFor`, `offerGateSql`).

**Inputs.** Every product's listings (status and note, so customers' results count as much as the pilots'), the `live_checks` row of each listing (found = seen live) and live-run counts from `runs`. A person's note on a directory they switched off by hand (not the runtime's "disabled by live run") counts as evidence for offline and paywall.

**The four verdicts** (owner's rules, 4 Nov 2026):

| Verdict | When | Reasons (`reason` column), in plan order |
|---|---|---|
| **proven** | Accepted for at least one product: submitted, submitted_unconfirmed, approved, already_listed, badge_pending. | `live` (seen live for at least one product), `accepted`, `unconfirmed` (every acceptance was sent but never confirmed). Inside each: more products accepted first. |
| **badge** | Not proven, and the free listing needs a badge or backlink: needs_badge_block, badge_required, or a paywall, refusal or "complex" reading whose free route is a badge, backlink or reciprocal link. | `badge` |
| **trying** | Not proven, nothing rules it out. Includes captcha and sign-in-wall directories. | `pending`, `untried`, `fixable` (our form misses), `no_form` (no form found yet), `captcha` (captcha or human check not cleared yet), `free_closed` and `paywall_doubt` (the free option is full or paused, or a free route was seen beside a paid one), `dead_partial`, `declined`, then `blocked`, `phone`, `inbox_refused`, `mail_only`. The reason is the one most products share; a tie goes to the more promising one. |
| **excluded** | Only three facts about the directory itself: | `paywall`: a run read a paid-only page, no run saw any free route (full, paused, "for free", a free plan), nothing was accepted or badge-gated, nothing is in progress. `offline` / `not_found`: offline, or the submit page not found, for every product that tried (at least 2 agreeing), or a person noted it offline when switching it off. `not_products`: a community, a sales or advertising inquiry form, documentation, or "product submissions are excluded", with every product that tried finding no form. |

| ID | Rule | Trap | Code | Status |
|---|---|---|---|---|
| OFF-1 | Captcha and badge directories are never excluded; only a real paywall (no free option at all), a dead site or a site that takes no products is. | Owner's rule. The old offer page (`classify.py`) listed 43 captcha directories as unofferable. | `offer.ts` judge | Codified |
| OFF-2 | No verdict filters by product type: a refusal that names a product type (AI only, open source only, iOS only, design tools only) is `trying` / `declined`, so products that fit still get the directory. | Owner's rule (no AI-only or software-only gating). The old page listed 8 declined directories as unofferable. | `offer.ts` signal, judge | Codified |
| OFF-3 | A paywall is real only when no run on the directory saw a free route; "free queue full", "0 free slots" and "free plan paused" are `free_closed` (trying). | best-ai.org's "Sign in for free submission", aicorner.net's "free listing's Submit Tool button", stellarlaunch.org's "0 free slots" were all read as paywalls by one run. | `offer.ts` FREE_CLOSED, FREE_PATH, NO_FREE | Codified |
| OFF-4 | Dead means the site or its submit page, for every product that tried; a broken sign-up or API behind a page that loads is trying. Engine errors ("Target page, context or browser has been closed") are never dead. | `classify.py` counted `error` listings as offline; wallof.tools' sign-up answering HTTP 500 is not a dead site. | `offer.ts` DEAD_TEXT, ACCOUNT_BACKEND | Codified |
| OFF-5 | An acceptance anywhere wins over later negatives (proven), and the detail records "another run met a paywall". | tipseason.com was approved and live for Ad Studio, then its free queue closed. | `offer.ts` judge | Codified |
| OFF-6 | Paid plans (a product with an order) get every verdict but excluded, ordered proven, badge, trying, then by reason rank, then fit tags, products accepted, wave and authority. Products without an order (the pilots, admin products) get every directory, excluded ones last, so verdicts keep learning. | Owner's rule: ordering, not gating. | `plans.ts` PLANS.offers, offersFor; `index.ts` buildPlan | Codified |
| OFF-7 | A directory that turns excluded after a paid plan was built is never queued for that plan. | Verdicts move daily; a plan lives 48 h. | `index.ts` nextLive, dripGoogle (`offerGateSql`) | Codified |
| OFF-8 | `enabled = 0` still keeps a directory out of every plan, whatever its verdict. | Runtime switches a directory off on one paywall reading (CAT-6): 7 proven and 5 badge directories are off today (tipseason, ainavhub, launchpanda, viesearch, ebool, euroalternative, stackledge; gets.tools, agentwork.tools, launchranked, iqdir, toolso.ai). | `index.ts` buildPlan, nextLive | Codified; switching them back on is a person's call (Directories view shows verdict and state side by side) |
| OFF-9 | A person's verdict holds: `PATCH /api/admin/offers/:id` with `{offer, reason}` locks it; `{unlock: true}` hands it back. Editing the submit URL of a directory excluded as `offline`, `not_found` or `not_products` sets it to trying and locks it (the old evidence was about the old URL); unlock once runs on the new URL have a result. | Otherwise an excluded directory never gets new evidence from paid plans. | `offer.ts` setOffer, submitUrlChanged; `admin.ts` | Codified |
| OFF-10 | Customers see the verdict value only (`offer` on each listing, "track_record" column in the CSV: "Proven: has accepted our listings before", "Needs your badge block", "New: not proven yet"); reason and detail are admin only. | STA-5: notes carry engine internals. | `orders.ts` LISTING_FIELDS, reportCsv; `offer.ts` OFFER_LABEL | Codified |
| OFF-11 | The offer page reads the stored verdicts, never its own rules. | `classify.py` and the plans could drift. | `research/offering/pull.sh`, `build.py` | Codified (rebuild and push by hand) |

**Recompute.** Cron step `offerCron` on the 2-minute tick runs `computeOffers` once a day (an hour after a failed run); a claim row in `offer_runs` keeps two overlapping ticks from both running it. A run reads about 1,000 directories and 2,600 listings and takes about 15 s. Each run is logged in `offer_runs` (who, when, counts, directories that moved). Per row: `prev_offer` and `changed_at` record the last move.

**Admin endpoints** (behind `x-ocl-key`; the view key may GET):
- `GET /api/admin/offers` (optional `?offer=excluded`, `?reason=paywall`): counts per verdict and reason (all and enabled), the last 5 runs, and every directory with its verdict, reason, evidence, counts and lock.
- `POST /api/admin/offers/recompute`: recompute now; returns counts and how many moved.
- `PATCH /api/admin/offers/:id`: a person's verdict (`{offer, reason}`) or `{unlock: true}`.
- `GET /api/admin/directories` and `GET /api/products/:id/plan` carry `offer` and `offer_reason`; `GET /api/products/:id/listings` carries `offer`.

**Where it shows.** Admin Directories view (Verdict column, evidence on hover), plan items, listings, the customer dashboard JSON and CSV (OFF-10), the landing preview (counts leave excluded out; sample directories are proven ones) and the [offer page](https://farooqchisty-hub.github.io/oneclick-launch-build/offering.html) (`zsh research/offering/pull.sh <dir>; python3 research/offering/build.py <dir> <out>`, then copy `offering.html` and `offering.csv` into the build-doc repo and push).

**Scale note.** `computeOffers` reads every listing in one query. Past roughly 100 customers (100,000 listings) page it by directory.

## 1. Catalog and plan

| ID | Rule | Trap | Code | Status |
|---|---|---|---|---|
| CAT-1 | Every product is planned on every enabled directory in its segments, niche ones included; fit tags only order niche directories, they never exclude. A paid plan leaves out only directories whose verdict is excluded (OFF-6). | Segment filters left products on a fraction of the catalog; the promise is the submission, the directory decides. | `index.ts` buildPlan, segmentsFor, fitsFor; `plans.ts` offersFor | Codified |
| CAT-2 | Plans run by verdict (proven, badge, trying, excluded last for pilots), then the verdict's reason rank, fit tags, products accepted, wave, then authority (DR) high first. | Low-value directories used up the first day's capacity; unproven ones ran before proven ones. | `index.ts` buildPlan | Codified |
| CAT-3 | Lane G is planned only when the order's plan includes it (`google_lane`) or the product has no order, and runs only on a ready, unpaused Google account. | Products without a Google account queued lane G work that could never run. | `index.ts` GOOGLE_PLAN_OK, nextLive | Codified |
| CAT-4 | The plan limit must exceed the eligible catalog. | A 150 cap ran the drip dry at 89 tried. | `index.ts` launchPaid (limit 1,000) | Codified; see gap 13 |
| CAT-5 | The drip skips directories whose last plan run for this product ended paywall, not_found, no_form or other, and blocked or captcha_gate when Kernel is off. | Dead directories were retried every tick. | `index.ts` nextLive | Codified |
| CAT-6 | A paywall verdict takes a directory out of every plan, but only after one second look: a checkout our own click opened, or a verdict that names a free option, goes back for the free route. | "No payment is required" success page disabled as a paywall; CostBench "CONTINUE TO PAYMENT: $149" pressed beside its free review. | `live.ts` runLive (paywall re-check), PAY_HOST, NOT_PAID | Codified; one product's verdict still disables for all. The offer verdict reads past it (a directory accepted elsewhere stays proven, OFF-5), but `enabled = 0` still keeps it out of plans (OFF-8) |
| CAT-7 | A directory is switched off only after 3 different products miss it, never for lanes B, C or G, and switched back on when a run gets through. | One miss on any product had dropped 268 directories (40 had worked elsewhere); lane G misses were the shared Google account (24 lost). | `live.ts` runLive (miss rule) | Codified, being replaced (gap 2) |
| CAT-8 | Paid plans exclude only directories whose verdict is excluded (a real paywall, a dead site, not a place for products). | 172 "unofferable" directories were enabled and planned; most were captcha, no-form and our own misses, which the owner keeps in plans. | `offer.ts`; `plans.ts`; `index.ts` buildPlan, nextLive | Codified 4 Nov (gap 1) |
| CAT-9 | Catalog imports exclude GPT-only, DR-required, built-with, integration-marketplace and phone-required directories; badge-required ones are included (lane C). | These never accepted a listing. | research `import_new.py` EXCLUDE sets (outside the Worker) | Documented |
| CAT-10 | An Always On renewal adds directories new to the catalog without touching planned or finished ones. | Replans must not reset progress. | `orders.ts` renewed, `index.ts` buildPlan (INSERT OR IGNORE) | Codified |
| CAT-11 | The offer page is rebuilt from the engine's stored verdicts with `pull.sh` and `build.py` (`classify.py` is retired) and pushed to the build-doc site. | Offer claims must match proof per pilot, and the page must match the plans. | `research/offering/` | Codified source; rebuild and push by hand |
| CAT-12 | A directory's lane changes only on evidence from more than one run. | One run that reads a page as Google-only moves the directory to lane G for every product (3 moved; 2 had been accepted by email, and the canary caught the click that causes it, FORM-22). | `live.ts` runLive (move to lane G) | Gap 6 |

## 2. Crawl and kit

| ID | Rule | Trap | Code | Status |
|---|---|---|---|---|
| KIT-1 | The kit uses only crawled facts (null when not stated), no hype words, no em or en dashes, respects every character limit, and its variants differ in angle. | Invented founders, years and prices; identical copy on every directory. | `profile.ts` KIT_SYSTEM | Codified |
| KIT-2 | Crawl the home page plus up to 6 same-site pages (pricing, about, features, FAQ, legal). | Home pages alone lacked pricing and company facts. | `profile.ts` runProfile, EXTRA_PAGES | Codified |
| KIT-3 | The logo is rendered once to a 512 px transparent PNG and reused everywhere. | Most directories refuse SVG logos. | `dryrun.ts` logoPng | Codified |
| KIT-4 | Screenshots upload as JPEG (about 100 KB, at most 1,280 wide); a multi-file input gets desktop, wide and mobile under 600 KB together. | 220 to 520 KB PNGs broke "less than 200 KB"; three PNGs went over the CDP 1 MiB frame and closed the browser mid-run. | `dryrun.ts` shotJpeg, applyMappings | Codified |
| KIT-5 | Text variants rotate per product so two customers never send the same copy to one directory. | Duplicate copy across customers. | `recipe.ts` pickVariant, seedOf | Codified |
| KIT-6 | Links carry `utm_source=<directory-with-dashes>`, `utm_medium=directory`, `utm_campaign=launch`; a site that refuses query strings or long URLs gets the bare URL; maxlength cuts the query before any character. | URL fields rejected tracked links. | `live.ts` trackedUrl, URL_REFUSED; `dryrun.ts` fitLength | Codified |
| KIT-7 | The customer may edit the kit; it counts as approved 10 minutes after payment; edits apply to later runs. | The kit must never block the launch. | `orders.ts` settleKit, KIT_EDITABLE | Codified |
| KIT-8 | A site the plain fetch cannot read still gets a kit. | Script-only or bot-walled sites end `error` and the paid order never launches. | `profile.ts` runProfile | Gap 11 |

## 3. Account creation and inbox

| ID | Rule | Trap | Code | Status |
|---|---|---|---|---|
| ACC-1 | One identity per product and directory: `<slug>.<dircode>@<INBOX_DOMAIN>`, unique, password sealed with AES-GCM; product slugs are unique. | Shared addresses would route mail to the wrong product. | `live.ts` identity; `vault.ts`; `index.ts` setLive | Codified |
| ACC-2 | The inbox is a catch-all: unknown recipients are rejected; a +alias falls back to its base address. | Sites that lost our first account needed a fresh address that still lands here. | `inbox.ts` handleEmail | Codified |
| ACC-3 | Verification links and codes are used inside the run's own browser session, waited for in the run (75 s); the inbox never fetches them. | "Confirm Your Signup" and "NNNNNN is your sign-in code" mails were classed other; a plain fetch burns one-time links. | `live.ts` waitForMail, usableMail; `inbox.ts` handleEmail | Codified |
| ACC-4 | The link to open: an on-site verify link, then the auth provider's (Supabase, Clerk, Firebase oobCode, a tracked redirect), never a bare homepage. | The homepage was opened before an off-site Supabase link. | `live.ts` pickLink | Codified |
| ACC-5 | Verify pages that wait for a click get the click. | TinyLaunch /auth/verify holds the link until a button press (anti-scanner). | `live.ts` runLive (verify open) | Codified |
| ACC-6 | Codes: from the subject first, then near "code"; one box or one box per digit, typed from the first box; press a button only if the code box is still there; codes older than 10 minutes are expired. | Widgets that move focus ignored fill on later boxes; screens that submit on the last digit were submitted twice. | `live.ts` enterCode; `inbox.ts` codeOf | Codified |
| ACC-7 | A resend voids earlier links: older mail is never opened after one. | Expired links opened after a resend. | `live.ts` runLive (resend) | Codified |
| ACC-8 | Two-step sign-in (email, then password or code) is not signed in until the screen changes. | The password screen was treated as signed in. | `live.ts` runLive (two-step) | Codified |
| ACC-9 | A refused sign-in goes through the site's own Forgot password in the same session; with no reset path, sign up afresh; an address already taken gets a +alias account. | Laravel, WordPress, NextAuth `?error=CredentialsSignin` and French Hotfrog refusals were retried with the same password until stuck. | `live.ts` WRONG_CREDS, runLive (reset, alias) | Codified |
| ACC-10 | A sign-in answered with nothing twice, every field filled, means the account never existed: reset, then sign up. | Silent sign-ins looped. | `live.ts` runLive (silent sign-in) | Codified |
| ACC-11 | "Already have an account? Log in" is a link, not a refusal. | Accounts were marked created that never were; every later run signed in to nothing. | `live.ts` ALREADY, EXISTS | Codified |
| ACC-12 | Find the sign-up path from visible links only; with none, try the usual /register and /signup paths once; a confirm-password box makes a page a sign-up form. | A hidden mobile-menu "Register" timed out every click; b2saas.com's page said nothing else. | `live.ts` guessSignup | Codified |
| ACC-13 | Lanes A, B and C never press OAuth buttons in the form; a page the model reads as Google-only moves the directory to lane G. | OAuth popups stalled runs. | `live.ts` OAUTH_BTN, GOOGLE_ONLY, runLive (move to lane G) | Codified; the move on one verdict is CAT-12, and form opening can still press one (FORM-22) |
| ACC-14 | A code sent to a phone (SMS, WhatsApp, call) skips the directory; never wait or guess. | Owner's rule; "does not request SMS" wording is not one. | `live.ts` PHONE_OTP | Codified |
| ACC-15 | Every email box gets the directory's inbox address, never a person's address. | Verify links must stay automatable. | `live.ts` runLive (email boxes), fillEmptyRequired | Codified |
| ACC-16 | Mail reruns: one run per new verification mail (short delay so a burst shares it), at most 6 runs a day per listing (3 on lane G); one timed re-check 2 h on, at most daily, never lane G. | A site that mails on every visit kept a listing re-running. | `inbox.ts` mailKick, mailRunsPerDay; `live.ts` runLive (follow-ups) | Codified |
| ACC-17 | Only a real "you are live / approved / published" mail marks a listing approved; a receipt keeps it submitted; in doubt, a receipt. | "Submission received" read as approval. | `inbox.ts` LIVE_MAIL, NOT_YET, KEEP_STATUS | Codified |
| ACC-18 | Mail click trackers are never opened; a wrapper's inner address is used. | A tracker can hide an unsubscribe or confirm action. | `inbox.ts` listingLinks; `liveness.ts` isTracker, unwrap | Codified |
| ACC-19 | Directory mail reaches the customer as a daily digest or a forward; verification mail never does. | Customers must not act on our account mail. | `notify.ts` sendDigests, forwardMail | Codified (needs BREVO_API_KEY) |
| ACC-20 | Generated passwords fit the site's limit. | startup.info accepts at most 15 characters; ours are 20. | `vault.ts` newPassword | Gap 12 |
| ACC-21 | Notes and stored text never carry em or en dashes. | Owner's rule; notes quote site and mail text. | `inbox.ts` handleEmail; `gaccounts.ts` setNote | Codified |

## 4. Google lane

| ID | Rule | Trap | Code | Status |
|---|---|---|---|---|
| GOO-1 | Each Google account lives in one long-lived Kernel stealth browser, signed in once by a person in the live view; runs attach to it as tabs. | Google binds a sign-in to its browser: copied profiles came back signed out. | `kernel.ts` googleLiveStart, kernelAttach; `gaccounts.ts` | Codified |
| GOO-2 | Attach with `persistent: true`, to the browser's own context. | Without it Playwright drove a fresh empty context: no cookies, no solver. | `kernel.ts` kernelSession | Codified |
| GOO-3 | Never create Google accounts or loop sign-in retries; use older real Gmail accounts from a pool. | Fresh Cloud Identity accounts were hard-rejected; about 8 automated retries in an hour locked two accounts. | `gaccounts.ts` (pool); `gidentity.ts` unused | Documented |
| GOO-4 | "Verify it's you" is answered by the account's software passkey (CDP virtual authenticator); a password is never typed; on the password page use "Try another way", never Next. | Per-directory OAuth triggered phone-tap prompts; Next on the password page submits an empty password. | `kernel.ts` googleSteps, keyTab, passkeyCredential | Codified |
| GOO-5 | The passkey's sign counter only rises: loaded at least at the seconds since 1 Nov 2026 and saved after every assertion. | Runs that ended before saving left Google holding a higher count. | `kernel.ts` passkeyCredential, googleFinish | Codified |
| GOO-6 | The account chooser picks only the pool account's email; if it is not listed, "Use another account" and its address; a screen for another account ends the run and never pauses the lane. | 2 of 144 runs clicked a person's own Gmail left in the chooser from setup; the passkey could not answer, Google showed an error, the engine read a challenge and paused the lane 6 h each time. | `kernel.ts` googleSteps (chooser); `gaccounts.ts` accountAuth | Codified (deployed 4 Nov 13:27 UTC) |
| GOO-7 | Google's after-sign-in prompts (selfie video, recovery phone, create a passkey) are always skipped; consent is never pressed on a passkey page. | Their Continue starts the selfie flow or makes a new passkey. | `kernel.ts` googleSteps | Codified |
| GOO-8 | The Google button is found by its own words, aria label or Google's button frame, never a plain link that mentions Google; follow the site's own Sign in when the button sits behind it. | "Google Search Preview" tools; arcapush's button behind "Sign in to continue". | `kernel.ts` googleButton, googleSignIn | Codified |
| GOO-9 | Google screens are judged by host and parameters, and only after the page settles. | The selfie prompt lives on myaccount.google.com; the URL trails the page during the hand-off to the site. | `kernel.ts` googleScreen, googleSteps | Codified |
| GOO-10 | A Google screen is never a result. | youtools.ai was marked submitted_unconfirmed three times on "Choose an account". | `live.ts` runLive (proof) | Codified |
| GOO-11 | Pacing per account: one sign-in at a time, `GOOGLE_GAP_MIN` (5) apart, `GOOGLE_DAY_CAP` (80) a day, 6 h pause after a real Google challenge; at the cap, wait only until the window frees. | A flat 6 h wait pushed the drip past the moment the cap freed. | `live.ts` runLive (Google pacing); `gaccounts.ts` pauseAccount | Codified |
| GOO-12 | Only runs that really went through Google sign-in count for pacing and retries. | "No google button" runs used up sign-in budget and retry caps. | `live.ts` runLive (lane marker rename) | Codified |
| GOO-13 | One lane G listing in flight per account; never-tried items first (products take turns), then retries idle 2 h, at most 2 lane G runs per listing. | Retries starved first attempts. | `index.ts` dripGoogle | Codified |
| GOO-14 | The shared Google browser closes only this run's tabs and is never held for a human check. | Runs closed each other's tabs. | `live.ts` runLive (closeOwnTabs) | Codified |
| GOO-15 | Every Google sign-in leaves from one static ISP exit IP. | Changing exits looked like new devices. | `kernel.ts` googleLiveStart (GOOGLE_PROXY_ID) | Codified |
| GOO-16 | A customer's own Google account comes off at delivery: our passkey deleted, signed out, browser deleted. | We must not keep access past the launch. | `gaccounts.ts` removeJob, releaseCustomerGoogle | Codified |
| GOO-17 | Registering a passkey needs a person to approve it on a device that already has one. | Google holds new passkeys for a security delay. | `gaccounts.ts` passkeyJob | Documented (manual step) |
| GOO-18 | Pool capacity matches the lane deadline. | One account, 80 sign-ins a day, 264 lane G directories per product. | `gaccounts.ts` pickPool | Gap 7 |

## 5. Page reading and field filling

| ID | Rule | Trap | Code | Status |
|---|---|---|---|---|
| FORM-1 | Locate fields by their own id or name, never React-generated ids; visible copies only; after the model call, re-tag the page so mappings still point at the right box. | Hidden duplicates and honeypots share names; forms re-rendered during the 10 to 60 s model call and every tagged field timed out. | `dryrun.ts` fieldLocator, GENERATED_ID, retag | Codified |
| FORM-2 | Custom widgets are fields: the combobox (not the hidden native select), styled radios and checkboxes ("Group: Option"), button radios, chips, popover multi-selects, option cards, Google Forms dropdowns, pickers inside closed `<details>`. | Each one was invisible to the model, and required groups stayed empty. | `dryrun.ts` collectOnce, inspect | Codified |
| FORM-3 | Fill by the field's real kind, not the model's verb. | "select" on a custom dropdown threw; "choose" on a checkbox ticked it on and off. | `dryrun.ts` applyMappings | Codified |
| FORM-4 | Tag boxes get items one at a time (their + button or Enter). | A comma-joined string left "Please add at least one keyword". | `dryrun.ts` addTags | Codified |
| FORM-5 | Typeahead boxes: click the closest new suggestion under the box; retype key by key if none opens. | Hotfrog's category box keeps nothing until a suggestion is clicked. | `dryrun.ts` pickSuggestion | Codified |
| FORM-6 | Options: exact, then fuzzy, then a generic fit; never paid, featured or sponsored; a required "select at least one" beats an empty group. | Near-miss option names left required selects on the placeholder. | `dryrun.ts` applyMappings; `live.ts` pickRequiredSelects, FALLBACK_OPTION | Codified |
| FORM-7 | Required fields are never left empty and never invented: missing links get the website URL, numbers "Not disclosed", country and size from the site (else United States, 1 to 10); optional fields with no fact stay empty. | Forms stalled on one empty required box; the owner's rule forbids invented facts. | `live.ts` LIVE_SYSTEM, fillEmptyRequired | Codified |
| FORM-8 | A minimum length is met with the long description or two variants joined. | "Please lengthen this text to 300 characters". | `live.ts` fillEmptyRequired | Codified |
| FORM-9 | Arithmetic anti-spam questions in a form are answered by rule, read again from the page each time. | They change on every load (aigcsoft.site "1 + 6 ="). | `live.ts` solveMath | Codified |
| FORM-10 | Newsletter and site-search forms are marked and never filled or pressed. | They share the "Submit" label with the real form. | `live.ts` markNewsletters, NOT_NEWS, SEARCH_ERR | Codified |
| FORM-11 | Promo and newsletter popups are closed; cookie banners get the privacy choice (decline non-essential), never accept; a modal holding a real form or a password box is never closed. | Popups swallowed the submit click; Hotfrog's banner covered fields. | `live.ts` dismissPopups | Codified |
| FORM-12 | Required terms and consent boxes are ticked by rule; marketing opt-ins never. | Forms refused without consent. | `dryrun.ts` autoConsent | Codified |
| FORM-13 | Plan pickers: press the free card; a paused, full or sold-out free plan is not an option; a checkout button is never pressed. | stackledge.com "Free plan paused"; /get-started sites submit nothing until a plan is picked. | `live.ts` pickFreePlan, PAID_BUTTON | Codified |
| FORM-14 | The submit is a button inside the form holding the filled fields, never a tab, toggle or nav link with the same words. | LeanVibe's "Log In" tab was pressed instead of its "Log In" button, swapping in an empty form. | `live.ts` pickSubmit, clickSubmit | Codified |
| FORM-15 | A disabled submit is explained, not forced: wait for uploads, fetch an inline email code, send a captcha to the stealth browser. | Enter pressed elsewhere did nothing. | `live.ts` runLive (disabled button) | Codified |
| FORM-16 | Step budget: 7 steps plus one per new screen (at most 14); one form gets 3 tries, up to 6 when each try draws a different complaint. | Long wizards ran out of steps; stuck forms looped. | `live.ts` runLive (step budget) | Codified |
| FORM-17 | A submit URL that shows a landing or pricing page: follow up to two same-site "submit" or "free" clicks; after signing in, reopen the submit page once, or use the dashboard's own "Add listing" when the submit URL is the sign-up page. | AIToolboard's /pricing landed signed-in users on a profile; sign-up submit URLs looped. | `dryrun.ts` openForm, followClick; `live.ts` runLive (reopen) | Codified |
| FORM-18 | A proven recipe is replayed step by step without the model; only steps that fit the account's state (never a sign-up for an account we made). | Replays re-ran sign-up for existing identities. | `recipe.ts` replay; `live.ts` runLive (replay) | Codified |
| FORM-19 | Only accepted flows become recipes. | 468 recipes come from awaiting_email runs; 55 directories have one and no acceptance. | `live.ts` runLive (recipe save) | Gap 10 |
| FORM-20 | The browser's own validation is read only for fields a person sees that belong to the listing; wording already on the page before the first submit is copy, not an error. | Footer newsletters and hidden login modals produced "Please fill out this field" noise. | `dryrun.ts` errorText | Codified |
| FORM-21 | Required phone boxes get the product's phone with the right dial code. | Dial code hard-coded to India; customers have no phone. | `live.ts` fillEmptyRequired, LIVE_SYSTEM | Gap 8 |
| FORM-22 | Opening the form never follows an OAuth button or a product card whose name merely starts with "Free". | Canary, 4 Nov: ainave.com pressed "Continue with Google" (ranked as "continue"); resource.fyi opened its "Free App Logo" product page. | `dryrun.ts` collectOnce (rank), openForm | Gap 6 |
| FORM-23 | Honeypots are never fields and never filled: a text box out of sight (off-screen, under aria-hidden, opacity 0, 1px) with tabindex -1, or one named or labelled as a trap ("Leave this empty"), is marked by collect; field locators skip it; a value the run typed into one is emptied. | madelaunch.com answered "Invalid submission." while its off-screen "website" box held the product URL; aitoolsmarketer.com, apitracker.io, agentscouthq.com, fewservers.com and costbench.com ("Company URL" at -10000px) carry the same kind of box. | `dryrun.ts` collectOnce (honeypot, data-ocl-hp), fieldLocator; `live.ts` fillEmptyRequired | Codified |
| FORM-24 | A select hidden behind Chosen or select2 is the field (with its options), not the widget's search box; it takes its option without the visibility check and the widget is told; an empty select the site's last complaint names counts as required. | topdesignking.com's colour, category and tag selects stayed empty and the server answered "Colour field is required." on every try. | `dryrun.ts` collectOnce, fieldLocator, applyMappings (select); `live.ts` pickRequiredSelects | Codified |
| FORM-25 | Chips: an option is the chip's last line with letters (icon names and counts dropped); a missing option opens the row's "Show all"; when none of the model's options is a chip, a picker row (required or labelled categories, tags, platforms) gets the closest generic chip, else the first, never a paid one. | junkstartups.com's chips read "ShoppingCart\nE-Commerce", the model asked for "Marketing", and the click waited 3 s and failed on every try (15 fill errors). | `dryrun.ts` collectOnce (chips), applyMappings (choose on chips) | Codified |
| FORM-26 | A searchable dropdown types only into a search box that is showing and is not an email, password or newsletter box; a typed word that empties the list is cleared and the option is picked from the full list. | toolplateai.io's react-select category got "no option like Marketing" (an option it has) after the word went into another dialog's box. | `dryrun.ts` applyMappings (choose) | Codified |
| FORM-27 | A form a directory embeds from a form builder (Google Forms, Tally, Typeform, Jotform, Airtable, Fillout and similar, in the page or in a modal a control opens) is opened at its own address. | spiff.store's free "Submit AI Tool" opened a Google Form in a modal and the page read showed no fields; feedmystartup.com, listoffreeware.com and frameplate.co embed theirs and ended no_form. | `dryrun.ts` embeddedForm, openForm; `live.ts` runLive (pressed control) | Codified |
| FORM-28 | A box the site's last complaint names (by its name, id, label or placeholder) is filled like a required one. | web-sites-directory.com's WordPress editor box has no label or required flag; the site said "Error: Post content required" on every try. | `live.ts` fillEmptyRequired (complaint) | Codified |
| FORM-29 | A screenshot box with a stated size beside it ("Please upload 1600X970 image", "Recommended: 1280x720px") gets the screenshot cut to exactly that size. | topdesignking.com refused the 1280-wide screenshot on every try. | `dryrun.ts` applyMappings (upload), shotJpeg (exact) | Codified |
| FORM-30 | A box claiming the directory's badge or a backlink is already on the product's site is never ticked by rule (consent and required-box fillers), and one the model ticked is unticked while the badge is not live; a required one that blocks the form ends needs_badge_block. | aitoolslist.tools: "I confirm that I have placed the selected dofollow badge or backlink on my website" (autoConsent ticked it as consent, a false claim). | `dryrun.ts` BADGE_CLAIM, autoConsent; `live.ts` fillEmptyRequired, runLive (same form) | Codified |
| FORM-31 | A run on its way to the sign-up form (state signing_up) tells the model "no account yet"; a page a verification link opens to finish the account (complete, set password, activate) is completed, never left for the sign-in page, and a run whose first act was opening that link starts there instead of the submit page. | aitoolex.com, digitool.co, launchpaddock.com, topaihubs.com, devpost.com, maqtoob.com: the sign-up form was read as a sign-in and its "Sign in" link pressed (15 listings); madelaunch.com's /complete-registration was left for sign-in, which answered "Your registration is incomplete". | `live.ts` runLive (model prompt, sign-in switch, useMail) | Codified |
| FORM-32 | The dry-run prompt reads like the live one on two borderline pages: a free or $0 tier beside a pre-selected paid tier is form_ready, and an email and password sign-in beside Google or GitHub buttons is login_required. | Canary 4 Nov: costbench.com read paywall once its honeypot left the field list; aijet.cc read "Google and GitHub only". | `dryrun.ts` MAP_SYSTEM | Codified |

## 6. Submit and success detection

| ID | Rule | Trap | Code | Status |
|---|---|---|---|---|
| SUB-1 | The outcome is read only from text that appeared after the click; a MutationObserver keeps toasts that vanish. | Static "under review" copy gave false successes; success banners on reset forms gave false misses. | `live.ts` freshText, watchMessages, readMessages | Codified |
| SUB-2 | Success wording is multilingual and never a bare "submitted". | AI SuperHub's "submitted email" was a $49 paywall; non-English receipts read as "did not advance". | `live.ts` SUCCESS | Codified |
| SUB-3 | Instructions ("By clicking submit...") and the form's own labels and options are never a result. | "Subscription or recurring payment required" (an option) was read as a paywall. | `live.ts` INSTRUCTION, runLive (form words) | Codified |
| SUB-4 | Upload, save and copy toasts are never receipts. | "Files uploaded successfully" between wizard steps. | `live.ts` NOT_RECEIPT | Codified |
| SUB-5 | Busy wording ("Uploading", "Loading") clears before a verdict. | Results judged mid-upload. | `live.ts` runLive (settle) | Codified |
| SUB-6 | A weak "we will be in touch" above 2 or more empty required boxes is a next step. | A Typeform question description read as a receipt. | `live.ts` runLive (Typeform) | Codified |
| SUB-7 | Wizard steps and sign-in pages reached by the submit click are never `submitted_unconfirmed`. | bai.tools' button led to /api/auth/signin and was counted as submitted. | `live.ts` runLive (landing checks) | Codified |
| SUB-8 | A script form that posted to its own endpoint (2xx, no error) and cleared every box, with no message, is `submitted_unconfirmed`; never for sign-in or sign-up forms. | aiforbusinesses.com and supahero.io came back blank three times and were called "required field empty". | `live.ts` runLive (cleared-form post) | Codified |
| SUB-9 | A message block shown again counts as the site's words; a typewriter animation keeps only its longest form. | Webflow's "Oops! Something went wrong" is the same block each time. | `live.ts` readMessages | Codified |
| SUB-10 | Already listed, declined and paywall-after-submit are detected from fresh text. | Each was read as a form miss. | `live.ts` ALREADY, DECLINED, PAYWALL | Codified |
| SUB-11 | An accepted listing is never overwritten or re-submitted by a later, duplicate or stale job. | An approval mail landed while a retry ran. | `live.ts` KEEP, runLive (start guards) | Codified |
| SUB-12 | Chrome's "not secure" warning on plain-http forms: send anyway. | Submits stalled on the interstitial. | `live.ts` INSECURE | Codified |
| SUB-13 | Links on the final screen are hints for the liveness check, never written as `listing_url`. | Dashboards and thank-you pages were stored as listings. | `live.ts` runLive (proof); `liveness.ts` hintListing | Codified |
| SUB-14 | "Ready!" and "Solving..." are the solver's badge, not the site. | Solver badge text read as a result. | `live.ts` runLive (fresh text) | Codified |
| SUB-15 | "Already submitted, add our badge to go live" is badge pending. | Read as already listed. | `live.ts` runLive (badge) | Codified |
| SUB-16 | The site's complaints are kept: every line of the last complaint still on the page counts again; a same-page answer with complaints over mostly the same fields is not a new step; a silent try reports the earlier complaint; the model sees every complaint the page drew in the run. | 70 listings ended "a required field is probably empty" although an earlier try had said which; webdirectory.top went back and forth between its title and description minimums. | `live.ts` runLive (said, whyNot, formComplaint, pageComplaints, COMPLAINT) | Codified |
| SUB-17 | A success toast on a page that shows a later step of the same wizard ("Step 2 of 3", a line starting "2 / 3") continues the run; a receipt line still ends it. That step is never a side form to leave for the submit page and never a quiet receipt. | aitoolex.com said "Successfully created" on its way to the plan step that publishes the listing. | `live.ts` runLive (nextStep) | Codified |
| SUB-18 | The product's own listing page after a submit is the receipt: the page names the product and links our tracked URL for this directory, with no preview, draft, plan, payment or confirm step left; then `submitted_unconfirmed`, also on a no-form page later in a run that sent a product form. | startupeurope.net's "Save" opened the new startup page and the run, reading "an existing listing", pressed "Submit a startup" again (all three pilots ended not_submitted). | `live.ts` runLive (ownListingShown) | Codified |
| SUB-19 | "Already been published on <directory>" (new text or the site's validation line) is already listed. | euroalternative.eu showed it beside the URL box and the run resubmitted. | `live.ts` ALREADY, runLive | Codified |
| SUB-20 | A screen saying nothing was sent ("Not sent yet", "send it by email", "Nothing has been confirmed as sent yet") is a mail-only directory: `complex`, never a submission. | fewservers.com was counted submitted_unconfirmed for all three pilots; apitracker.io's draft screen is the same. | `live.ts` NOT_SENT, runLive | Codified |

## 7. Kernel and captcha

| ID | Rule | Trap | Code | Status |
|---|---|---|---|---|
| KER-1 | **Captcha stance**: the engine uses Kernel's built-in captcha solver as configured and nothing beyond it: no custom solving, no solver services, no bypass. A person may do a check in the live view (hcheck). Sites Kernel cannot clear stay in every plan as `trying` / `captcha`, never excluded (owner's rule 4 Nov, OFF-1). Plain arithmetic questions in a form are form fields (FORM-9), not captchas. | Owner's rule. | `kernel.ts` kernelSession (stealth), solverWait; `hcheck.ts`; `offer.ts` | Codified |
| KER-2 | Pages open in the browser's own context, on the tab the live view shows. | The solver is an extension of that context; fresh contexts and background tabs never got it. | `kernel.ts` kernelSession; `live.ts` runLive | Codified |
| KER-3 | Wait for the solver up to 90 s: done when a token lands, every task it took ends, or it never picked one up. | Runs gave up while the solver was working. | `kernel.ts` solverWait, captchaSolved | Codified |
| KER-4 | Lane B and `needs_stealth` listings run in Kernel; captcha runs share `KERNEL_STEALTH_SLOTS` (7 of 10) so the Google lane always has room; slots older than 15 min are leaks; no free slot means come back later, never a browser without the solver. | Kernel's concurrency cap; directories spent on solverless browsers. | `live.ts` runLive (slots) | Codified |
| KER-5 | Invisible bot checks that say "one moment, try again" get the same submit pressed again in the stealth browser. | The solver finishes within seconds; leaving the screen lost the run. | `live.ts` BOT_WAIT | Codified |
| KER-6 | A captcha that refused the submit: in Browser Run, end `needs_stealth` (one stealth retry follows); in Kernel, give the solver a few tries, then a person. | Captcha directories were marked failed after one browser. | `live.ts` CAPTCHA_FAIL, runLive (captcha) | Codified |
| KER-7 | Browser Run at its concurrency cap: come back in 1 to 2 minutes, never a failure. | Cap hits were recorded as directory failures. | `live.ts` runLive (launch) | Codified |
| KER-8 | Workers Playwright attaches to Kernel only through a fake browser binding whose fetch opens Kernel's CDP socket. | `connect()` works only through bindings. | `kernel.ts` kernelSession | Codified |
| KER-9 | Visible Turnstile and hCaptcha were never taken by the solver over CDP: they end at a person or out of the offer. | Kernel telemetry showed zero solve starts on visible challenges. | `kernel.ts` solverWait (telemetry) | Documented |
| HC-1 | Human checks: fill every other field, then park the tab in the product's check browser; never park with required fields still empty (a person never fills those). | Parked forms came back with missing fields. | `hcheck.ts` parkCheck, missingRequired | Codified |
| HC-2 | One product per check browser (cookies never mix accounts), 12 tabs each, at most `CHECK_BROWSERS` (3) browsers, each holding a Kernel session from the stealth share. | A browser per listing exhausted Kernel sessions. | `hcheck.ts` claimCheckBrowser | Codified |
| HC-3 | A resume acts only on proof a person did it (a token from a visible widget, a typed code that stopped changing, or the page moved on) and never submits twice. | Script-refreshed tokens looked like a person's work. | `hcheck.ts` cleared, markCheck | Codified |
| HC-4 | A parked tab refills after 24 h, a check browser is recycled after 72 h, 3 refusals of the same form stop it, a lost tab is filled again (never a failure). | Stale sessions and codes on old tabs. | `hcheck.ts` PARK_TTL, MAX_AGE, pollChecks, checkBrowserGone | Codified |
| HC-5 | A parked check has an owner, an alert and an end. | 9 parked, 18 h old, 3 of 7 stealth slots held, nobody told. | none | Gap 4 |

## 8. Badge lane

| ID | Rule | Trap | Code | Status |
|---|---|---|---|---|
| BAD-1 | Submit every lane C directory: take the free option with the badge, put the product URL in backlink fields, never tick "badge added" unless the check found it. | Owner's rule: list them all, rejection later is fine; never fake the check. | `live.ts` LIVE_SYSTEM; `badges.ts` | Codified |
| BAD-2 | The customer pastes one footer block: plain server HTML (checkers do not run scripts), the directory's own embed code when it publishes one, kept up (most re-check). | Script-injected badges were not seen by checkers. | `badges.ts` badgeBlock | Codified |
| BAD-3 | The badge counts as up only when `verifyBadges` finds it on the live homepage; `badge_ok` at 90%. | Claims without proof. | `badges.ts` verifyBadges | Codified |
| BAD-4 | A hard badge gate waits for the block (`needs_badge_block`) and never disables the directory. | Badge gates were dropping directories. | `live.ts` runLive (badge gate) | Codified |
| BAD-5 | A success before the badge is up is `badge_pending` (accepted). | The customer's step keeps the listing. | `live.ts` runLive (proof) | Codified |
| BAD-6 | Pasting the block is the customer's step; directories that check a standalone page (17) need more than the footer. | Footer block covers about 64 of the badge directories. | `badges.ts` | Documented |

## 9. Liveness

| ID | Rule | Trap | Code | Status |
|---|---|---|---|---|
| LIV-1 | "Live" means `live_checks.state = 'found'`, never just accepted. | The old "accepted plus any stored URL" was about 40% false. | `productview.ts` LIVE_COL, LIVE_JOIN; `status.js` customerStatus | Codified |
| LIV-2 | Check soon after acceptance, then 1 h, 1 d, 3 d, 7 d, weekly; a found listing is revalidated monthly. | One-off checks missed slow approvals. | `liveness.ts` SCHEDULE, nextCheck | Codified |
| LIV-3 | Climb a ladder, cheapest first: stored URL and hints, the directory's learned URL pattern, sitemaps, the home page's detail pattern, new pages, a home page link, site search, Browser Run, a search-engine query. | Single-method checks missed most listings. | `liveness.ts` probe | Codified (search-engine step off: TREG_TOKEN unset) |
| LIV-4 | Accept only a public page on the directory about this product with evidence (a link to its site, our tracked link, our copy or its domain); never a list, comparison, search, dashboard, submit or thank-you page; a URL we built needs a link or our copy, not the name. | Search pages echo the query; "PushOwl by Brevo" is not Brevo. | `liveness.ts` judge, makeProducts, notListing | Codified |
| LIV-5 | Directories without detail pages count as live when a list entry carries our tracked link, or our name beside our copy. | Old link directories have no listing page. | `liveness.ts` listEntry | Codified |
| LIV-6 | A generic name gets one more site search by the product's domain; then only tracked-link entries count. | "Ad Studio" sank below page one. | `liveness.ts` probe (step 7) | Codified |
| LIV-7 | Pending pages ("isn't live yet", a title that is only a status) are not live. | Pending pages counted as live. | `liveness.ts` PENDING, PENDING_HEAD | Codified |
| LIV-8 | Drill down one level only, never into launch or review sub-pages. | Sub-pages read as listings. | `liveness.ts` probe | Codified |
| LIV-9 | A tick out of budget leaves the listing due; it is never marked missing on half a check. | Half checks wrote "missing". | `liveness.ts` runLiveness | Codified |
| LIV-10 | A verified page sets `listing_url` and promotes submitted to approved; statuses never move out of accepted; a stored URL that is clearly not a listing is cleared and logged. | Wrong URLs shown to customers. | `liveness.ts` record | Codified |
| LIV-11 | A "you are live" mail link that names the product is trusted when the directory blocks our checks. | Bot-walled directories could never be verified. | `liveness.ts` checkListingNow (trust) | Codified |
| LIV-12 | An admin's "marked listed" counts as found and is rechecked in a year. | Manual proofs were overwritten. | `admin.ts` adminApi (listed) | Codified |

## 10. Pacing, retries and sweeper

| ID | Rule | Trap | Code | Status |
|---|---|---|---|---|
| OPS-1 | Every cron tick: a run `running` for 10 minutes is requeued (99% finish within 7); after 6 interruptions it ends `error`; a parked listing goes back to wait for a person. | Killed runs stayed "running" forever. | `ops.ts` sweep | Codified |
| OPS-2 | Deploy when nothing is running: a deploy kills in-flight runs ("Target page, context or browser has been closed"); the sweeper heals them. | Deploys mid-run cost attempts. | Deploy procedure above | Documented |
| OPS-3 | One stealth retry for a miss that might be the browser (error, no_submit_button, not_submitted), 2 h later, attempts under 3, 3 per tick, never lane G. | Browser Run misses on bot-protected sites. | `ops.ts` sweep | Codified |
| OPS-4 | At most 6 run starts per directory per hour across all customers. | Bursts from our IPs and inbox domain. | `ops.ts` directoryBusy | Codified |
| OPS-5 | Drip every 2 minutes: up to `PER_TICK` (30) per product per tick, `DAILY_LIVE` (1,500) per product per UTC day. | Slow drips missed deadlines. | `index.ts` drip; `wrangler.jsonc` | Codified |
| OPS-6 | Queue jobs run one per message, 30 at once, and are acknowledged on failure, never retried by the queue. | Queue retries repeated browser spend. | `index.ts` queue; `wrangler.jsonc` | Codified |
| OPS-7 | Model calls: JSON schema, validation, one corrective retry, then the fallback model; all on OpenRouter; alert under $5 credit. | Malformed JSON; provider errors; Replicate throttled to 60 a minute under $10 credit. | `llm.ts` structured; `alerts.ts` openrouterCredit | Codified |
| OPS-8 | Dry runs block every non-GET and every submit; they also write a run row, a draft recipe (never replayed) and the product's `plan_items.last_run`. | Dry runs must never submit; a dry outcome can make the drip skip that directory for that product. | `dryrun.ts` runDry | Codified |
| OPS-9 | Read the run's step trail (`runs.detail.log`) before guessing from screenshots. | Misdiagnosed misses. | `admin.ts` productRuns | Documented |
| OPS-10 | The canary passes after every deploy. | No regression check existed. | `scripts/canary.sh` | Codified as a script; not automatic (gap 3) |
| OPS-11 | VIEW_TOKEN is read-only: GET only, no live Google windows, no stored credentials. | Team access must not expose accounts. | `index.ts` access | Codified |

## 11. Statuses and customer reporting

| ID | Rule | Trap | Code | Status |
|---|---|---|---|---|
| STA-1 | Accepted = submitted, submitted_unconfirmed, approved, already_listed, badge_pending. | Counts differed by view. | `liveness.ts` ACCEPTED (and 7 copies) | Codified; copies disagree (gap 9) |
| STA-2 | Live = liveness found it (LIV-1). | Accepted was sold as live. | `productview.ts` | Codified |
| STA-3 | Customer states: Listed (accepted and live), Submitted in review (accepted, not seen yet), Badge pending, In progress (queued, running, awaiting_email, needs_check, and retryable outcomes before the lane deadline), Couldn't list (hard stops at once, retryable ones after the deadline). | Raw statuses confused customers. | `public/app/js/status.js` customerStatus; `productview.ts` customerState | Codified |
| STA-4 | Lane deadlines: A, B, C 24 h; G 48 h. | Retryable outcomes need an end. | `status.js` LANES | Codified |
| STA-5 | Customers never see notes or engine internals; the CSV is in their words. | Notes carry engine detail. | `orders.ts` customerOrder, reportCsv | Codified |
| STA-6 | Delivery at 48 h after payment, or earlier when nothing is open and the plan has nothing left; never early while a Google-lane plan waits for the customer's Google. | Early delivery cut plans short. | `orders.ts` deliverOrders | Codified |
| STA-7 | The only refund: fewer accepted listings than the plan guarantees by 48 h; raised as a high alert for a person to refund in full. A full refund stops the drip. | Owner's refund rule. | `orders.ts` refundCheck, refunded | Codified (refund itself manual) |
| STA-8 | `needs_check` ends. | It stays "In progress" forever. | `status.js` ACTIVE | Gap 4 |
| STA-9 | Each listing a customer sees carries its directory's track record (`offer`: proven, badge, trying), and the CSV has a `track_record` column in their words; never the verdict's reason or evidence. | A customer reading "Couldn't list" should see whether the directory had ever accepted us. | `orders.ts` LISTING_FIELDS, reportCsv; `offer.ts` OFFER_LABEL | Codified (the dashboard does not render it yet) |

## 12. Alerts

| ID | Rule | Trap | Code | Status |
|---|---|---|---|---|
| ALR-1 | One alert row per condition (dedupe key), resolved when it clears; a detector that could not check resolves nothing. | Duplicate and flapping alerts. | `alerts.ts` raise, scanAlerts | Codified |
| ALR-2 | Detectors: runs stuck over 15 min, Google lane paused or needing the customer, failure clusters (same normalized note on 5+ directories in 24 h), Kernel errors and a dead Google browser, OpenRouter credit, SLA at 24 h (`SLA_MIN_24H`, 100) and 48 h, refund owed. | Each was found by hand first. | `alerts.ts` DETECTORS; `orders.ts` refundCheck | Codified |
| ALR-3 | High alerts reach a person. | `ALERT_WEBHOOK` is not set: dashboard only. | `alerts.ts` scanAlerts | Gap 5 |
| ALR-4 | Alerts exist for an auto-disabled directory, a check parked over 2 h, a failed profile on a paid order and an acceptance drop after a deploy. | None of these are detected today. | none | Gap 5 |
