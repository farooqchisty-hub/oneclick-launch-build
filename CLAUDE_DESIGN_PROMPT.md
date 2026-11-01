# Prompt for Claude Design: landing page

Paste everything below the line into Claude Design. Iterate there; when it's locked, hand the export back and I'll wire it to the live preview, checkout and dashboard.

---

Design a high-converting landing page for a SaaS product. Working name: **LaunchKit** (placeholder; keep the name in one place so it can be swapped).

## What the product does (say it this simply)

You paste your product's URL and pay once. Our agent writes your listing, creates the accounts, clears the email checks, and submits your product to 150+ free directories (AI tool directories, SaaS and startup directories, launch boards, category directories). Most listings go out within 24 hours; the rest within 48. You get a dashboard with every listing, a proof screenshot of each submission, and a tracked link per directory so you can see which ones send traffic.

## Who it's for

Founders and small teams who just launched (or are relaunching) a SaaS, AI tool, app or developer tool, and know directories help with discovery, SEO and AI-search visibility but don't want to spend 40+ hours filling forms. They're skeptical of "submission services" that spam or overpromise. They decide fast and buy with a card.

## The one job of the page

Get the visitor to paste their URL into the preview box. The preview is the conversion engine: it shows THEIR product name and logo, how many directories they'd be submitted to, and 6 sample listings written from their own site, with no signup. From there, one click to checkout.

## Tone and copy rules

- Plain, specific, calm, confident. Short sentences. Talk like a founder to a founder.
- No hype words (revolutionary, game-changing, unleash, supercharge, seamless). No emojis.
- **Never use em dashes or en dashes anywhere** (use commas, colons or parentheses).
- Be honest about limits: we promise submission with proof, not approval, rankings or traffic. Directories review on their own schedule. This honesty is a trust feature: make it visible, not hidden.
- Prices in USD.

## Page structure (top to bottom)

1. **Nav:** logo, How it works, Proof, Pricing, FAQ, "Get listed" button.
2. **Hero:**
   - Headline idea: "Get your product on 150+ directories in 48 hours." Alternative to test: "Paste your URL. We list you on 150+ directories."
   - Subline: one sentence on what happens (we write, sign up, verify, submit, and show proof).
   - **URL input with a "See my directories" button**, the visual center of the hero.
   - Microcopy under it: "Free preview. No signup, no card."
   - Right side or below on mobile: the **preview panel**. Design all of its states:
     - **Empty:** an inviting example, e.g. a faded sample product.
     - **Loading:** a progress line ("Reading your site", "Matching directories", "Writing sample listings"), about 5 to 10 seconds.
     - **Result:** product logo and name, a big number "540 directories matched", a breakdown by type (open forms and email signup; forms with bot checks; free with a badge; Google sign-in), 6 sample listing cards (directory name, tagline, description), and the "Get listed for $49" button.
     - **Error:** "We couldn't open that site" and "We list software products only".
3. **Trust strip:** "Directories we submit to" with a row of directory names or logos (placeholders: 12 logo slots).
4. **Proof of work** (the most important section after the hero):
   - **Three case studies** as cards, each with a product logo, name and URL, "X directories accepted in 48 hours", a mini timeline (24h: N, 48h: N), a "See all listings" link, and 3 proof screenshot thumbnails.
     - Example 1: Ad Studio by Brevo, 134 directories accepted.
     - Examples 2 and 3: placeholders, "[Pilot product 2]" and "[Pilot product 3]", with "[N] directories".
   - A **live counter** row: total listings submitted, directories in our catalog, average time to done (placeholders: [N]).
   - **Proof screenshot gallery:** 6 placeholder frames labelled "Submission confirmation, [Directory name]", plus "Live listing" links (placeholder URLs).
5. **How it works:** 3 steps with simple illustrations.
   1. Paste your URL: we read your site and write your listing kit (taglines, descriptions, logo, screenshots); you can edit it, or let it auto-approve in 10 minutes.
   2. We submit: accounts, email verification and forms, done by our agent; you watch it in your dashboard.
   3. You get proof: every listing with a screenshot, a live link when the directory publishes it, and a tracked link.
6. **Dashboard preview:** a large mockup of the customer dashboard: a progress bar ("212 of 240 done"), a 24h and 48h countdown, a listings table (directory, status chip, proof, link), and a live feed ("Listed on Toolify 2 min ago"). Use placeholder data.
7. **What you get:** a checklist: 150+ submissions; a listing kit written from your site; proof screenshot per submission; UTM-tracked links; a final report (PDF and CSV); directory emails forwarded to you; an optional "Featured on" badge block for directories that ask for one.
8. **Honest caveats** (designed as a confident, friendly panel, not fine print):
   - "Directories approve on their own schedule, from minutes to weeks."
   - "Some ask for a badge on your site to keep the listing; we give you one block to paste, optional."
   - "We submit, we don't buy placements."
   - "We only list real software products."
9. **Comparison** (optional): "Doing it yourself" (40+ hours, 150 sign-ups, 150 verification emails) vs "Other submission services" (lists of links, no proof, unclear what was submitted) vs "LaunchKit" (done in 48h, proof for each, tracked links).
10. **Pricing:** one plan, one-time, $49 (placeholder). The features list, "Get listed" button, and a guarantee line placeholder: "[Guarantee: 100+ submissions or we refund the difference]".
11. **FAQ:**
    - Which directories?
    - Will they all approve me?
    - Do I need to do anything?
    - What about Google sign-in directories?
    - Is this spam?
    - Can I edit my listing?
    - What if my product isn't AI?
    - How is this different from launch kits that give you a list?
12. **Final CTA:** repeat the URL input.
13. **Footer:** Terms, Privacy, Contact, and "Made by [company]".

## Visual direction

- Not a template look. A confident, editorial product page with generous whitespace and one strong accent color.
- Mid-ground between a dev tool and a founder tool: think precise typography, real data on display, and UI mockups over illustrations.
- The preview panel and the proof section should feel alive (counts animating in, cards appearing one by one); respect reduced-motion settings.
- Light and dark themes.
- Mobile first: the hero URL input and preview must be excellent at 375px wide. No horizontal scroll.
- Fast: no heavy video, compressed images.

## Placeholders to use

Clearly mark every placeholder in brackets so it's easy to swap:
- [Pilot product 2 logo/name/url/count]
- [Proof screenshot: directory name]
- [Live listing URL]
- [Total listings submitted]
- [Average time to done]
- [Price]
- [Guarantee]
- [Company]

## Functional notes for the handoff (don't build the backend, just leave hooks)

- The preview calls `POST /api/preview {url}` and returns `{name, logo_url, counts: {automated, stealth, badge, google, total}, samples: [{directory, tagline, description}]}`. It's live today.
- Checkout calls `POST /api/checkout {url, email}` and returns `{checkout_url}` (Stripe). Collect the email in a small modal after "Get listed".
- Keep all copy in the HTML (no images of text) so I can edit it.

## Deliverables

- The full page at desktop (1440) and mobile (375), light and dark.
- All preview panel states.
- The dashboard mockup used on the page.
- An export (HTML/CSS) I can hand back for wiring.
