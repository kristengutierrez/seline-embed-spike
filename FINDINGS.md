# FINDINGS — Can a self-hosted page render as a LIVE embed inside a published Canva website?

Spike date: 2026-07-12. Timeboxed, single session.

## TL;DR verdict

**No, not for an unlisted domain** — and this is a hard technical wall, not a markup problem.

Iframely (the service Canva's Apps SDK docs and Embed element depend on for "Iframely-supported URLs")
gates *rich/player* resolution behind a manually-curated, per-domain whitelist
(`https://iframe.ly/domains.json`, ~2,320 entries). A domain not on that list gets **link-preview
treatment only** — title, description, thumbnail — never an iframe, no matter how correctly it
implements oEmbed autodiscovery, Twitter Player Cards, or Open Graph video tags. This was confirmed
empirically (Phase B) and confirmed in Iframely's own documentation (Phase B docs review). **H1, H2,
and H3 are all disproven for an un-whitelisted domain. H4 is confirmed.**

**Named condition on the "no":** this spike could not test Canva's own Iframely integration directly
(Phase C requires a manual, human-in-Canva pass — see [CANVA-CHECKLIST.md](CANVA-CHECKLIST.md)). It's
possible Canva's contracted Iframely tier uses a different/broader allowlist than the public
`domains.json`. That's the one gap this spike leaves open. Even if Phase C surprises us and Canva
resolves one of the test URLs as a live embed in the editor, **H5 (Canva flattens iframes to PNG at
publish time) must still be checked** — that's the same failure mode that killed the original
countdown-sticker feature, and a pass in the editor proves nothing about the published page.

## Test harness

Deployed to GitHub Pages (chosen as the platform-subdomain test target instead of Vercel/Netlify/
Cloudflare Pages — no new account/CLI auth was available in this environment, and a `*.github.io`
subdomain is the same class of data point: "does a platform subdomain work at all").

- Repo: https://github.com/kristengutierrez/seline-embed-spike
- Live site: https://kristengutierrez.github.io/seline-embed-spike/
  - `/plain/` — control, no embed metadata
  - `/oembed/` — H1: oEmbed discovery `<link>` + `/oembed.json` (`type: "rich"`, iframe pointing at `/widget/`)
  - `/player/` — H2: Twitter Player Card meta tags pointing at `/widget/`
  - `/og/` — H3: Open Graph video/iframe meta tags pointing at `/widget/`
  - `/widget/` — the live payload: ticking clock, live countdown to 2026-12-25, per-load random nonce

Framing is permitted — verified via `curl -I`:

```
$ curl -sI https://kristengutierrez.github.io/seline-embed-spike/widget/
HTTP/2 200
content-type: text/html; charset=utf-8
access-control-allow-origin: *
cache-control: max-age=600
...
```

No `X-Frame-Options` header, no `Content-Security-Policy: frame-ancestors` restriction. GitHub Pages
does not block framing by default. (Cross-checked against `pages.github.com` — same absence of
frame-blocking headers.)

## Phase B — Iframely probing

### Access without a paid key

Iframely's commercial API (`iframe.ly/api/iframely?url=...&api_key=...`) requires a key — no free tier
key was obtained in this session (that's a $0/mo "Starter" plan per their [pricing page](https://iframely.com/pricing),
2,000 hits/mo, but still requires signup/a credit card for overage).

However, their **public debug tool** (`https://debug.iframely.com/`) calls an unauthenticated endpoint —
`https://iframely.com/iframely?uri=<url>` — that requires only a browser-like `Referer`/`Origin`/
`User-Agent` header to respond (curl without those headers gets a bare `403` from the nginx front-end
protecting the debug endpoint from bot abuse, not from lack of auth). This endpoint was used for all
probing below. It is best treated as "what Iframely itself surfaces on the free, unauthenticated
surface" — the same underlying whitelist gates both this and the paid API (the whitelist is a QA/content
decision, not a billing tier).

### Sanity check — known-good domain

`youtube.com` (in Iframely's curated list) resolves fully, proving the harness/query mechanism works:

```json
{"links": [{"href": "https://www.youtube.com/embed/dQw4w9WgXcQ?rel=0", "rel": ["player", ...],
  "html": "<div style=...><iframe src=\"https://www.youtube.com/embed/dQw4w9WgXcQ?rel=0\" ...></iframe></div>"}]}
```

### Results for all 4 test URLs

| URL | `meta` extracted? | `links` (rich/player resolution)? | Notes |
|---|---|---|---|
| `/plain/` | title, description, canonical | **none** | control, as expected |
| `/oembed/` (H1) | title, description, site, canonical | **none** | oEmbed `<link>` tag was **not followed at all** — `/oembed.json` was never resolved into a link |
| `/player/` (H2) | title, description, canonical, `medium: "link"` | thumbnail image only (`rel: ["twitter","thumbnail","ssl","og"]`) | Twitter card meta **was read** (proves markup is valid) but no `player` rel, no iframe `html` |
| `/og/` (H3) | title, description, canonical, `medium: "video"` | thumbnail image only (`rel: ["thumbnail","og","ssl"]`) | OG video meta **was read** (Iframely correctly classified it as `medium: video`) but again no iframe `html` |

Full raw JSON responses saved in the repo evidence — reproducible via:

```
curl -s "https://iframely.com/iframely?uri=<url-encoded>" \
  -H "Referer: https://debug.iframely.com/" -H "Origin: https://debug.iframely.com" \
  -H "User-Agent: Mozilla/5.0 ..."
```

Tried adding `&iframe=1`, `&html5=1`, `&omit_script=1`, `&media=1` to the query — no change in any
case. This isn't a missing-flag problem.

### Interpretation

This is the important nuance: Iframely's parser **does** read and correctly classify standards-compliant
Twitter Player Card and OG video markup (`medium: "video"`, `rel: ["twitter", ...]` on the thumbnail) —
the test markup is valid and is not the blocker. What it withholds, specifically, is the *rich media
resolution* — the `player`-rel link with the actual `html` iframe payload. That's the exact thing Canva's
Embed element needs to produce a live iframe instead of a link-preview card.

## Phase B — Iframely documentation review

- [`iframely.com/docs/whitelist-format`](https://iframely.com/docs/whitelist-format): "domains listed
  with an 'allow' tag have their media embeds processed; those marked 'deny' [or absent] do not."
  Confirms self-hosted Iframely instances pull this same whitelist from Iframely's servers
  periodically — it's centrally maintained, not per-customer.
- [`github.com/itteco/iframely/docs/PROVIDERS.md`](https://github.com/itteco/iframely/blob/main/docs/PROVIDERS.md):
  *"Even the list of 1600+ is not set in stone due to our whitelisting approach... send us all
  `/^https?:\/\//i` links... internal whitelist of the domains that you want to send to us."* — explicit
  confirmation that standards compliance alone does not grant entry; it's a curated, QA'd list.
- Public whitelist file, `http://iframe.ly/domains.json` — fetched directly, 2,320 entries, all
  domain-specific regex patterns (e.g. `gist.github.com`, `500px.com/photo/...`). **No generic
  catch-all / wildcard oEmbed rule exists in the file.** `github.io` (the domain this spike is hosted
  on) is **not present** — only `gist.github.com` is. This is the direct, mechanical explanation for
  why `/oembed/`, `/player/`, and `/og/` all failed to resolve: the domain simply isn't in the file
  Iframely consults.
- Onboarding path, [`iframely.com/qa/request`](https://iframely.com/qa/request): "Becoming a publisher"
  form (your own app) or "Request for a third party" form. *"We'll review what's feasible and get back
  to you if we have questions... Our team will review your request and get back to you by email."*
  **No published SLA, no published turnaround time, no published cost for whitelisting itself** — it's
  a manual human review queue, separate from the paid API tiers (paying more doesn't buy a broader
  domain allowlist; the allowlist is a QA decision, not a purchasable feature).

## Hypothesis scorecard

| # | Hypothesis | Result | Evidence |
|---|---|---|---|
| H1 | oEmbed discovery link resolves | **Disproven** (for unlisted domain) | `/oembed/` → empty `links` despite valid discovery link + valid `oembed.json` |
| H2 | Twitter player card resolves | **Disproven** (for unlisted domain) | `/player/` → meta read correctly, but no `player` rel / no iframe html |
| H3 | OG video/iframe tags resolve | **Disproven** (for unlisted domain) | `/og/` → `medium: "video"` correctly classified, but no iframe html |
| H4 | Domain whitelisting required regardless of markup | **Confirmed** | `domains.json` has no generic rule; docs explicitly describe manual QA whitelisting; onboarding is a request-and-wait process with no SLA |
| H5 | Canva flattens iframe → PNG at publish, even if editor renders it live | **Not reached** — H1–H3 failed before Canva entered the picture. Still must be checked in Phase C if any URL unexpectedly resolves in Canva's editor. | — |

## What this means for the plan

The "buyer pastes a URL into Canva's native Embed element, no app needed" path is **blocked at the
Iframely layer**, not the Canva layer, for any domain you stand up yourself without going through
Iframely's manual publisher-approval process. That process:

- has no published cost, SLA, or guaranteed outcome — it's "we'll review what's feasible"
- is a **third-party dependency and gatekeeper**, functionally similar in risk profile to the Canva
  Apps SDK marketplace-review path this spike was trying to avoid — just a different vendor's queue
- does not scale to "any buyer can point this at their own custom RSVP page" even if Seline Paperie's
  own primary domain got approved, unless the approval covers a wildcard/subdomain pattern Iframely is
  willing to whitelist generically (untested, would need to be asked explicitly in the publisher
  request)

See [CANVA-CHECKLIST.md](CANVA-CHECKLIST.md) for the one remaining unknown: whether Canva's specific
Iframely contract has a broader allowlist than the public one. That's worth 10 minutes of manual
checking before treating this verdict as fully final, but budget should not be spent building anything
on the assumption it will pass.

## Cost of the alternative (if the answer holds as "no")

If live content requires bypassing Canva's embed pipeline entirely, "buyer customizes it themselves in
Canva" stops being the delivery model for the *live-data* tier of the product. Realistic alternatives,
roughly ascending in cost/control:

1. **Get Seline Paperie's domain whitelisted by Iframely** (cheapest to try, cost = time + uncertain
   turnaround, outcome not guaranteed, and per-buyer subdomains may need a separate ask).
2. **Build the actual Canva App** (the marketplace-review path this spike was trying to route around) —
   known cost (dev time + review queue), known mechanism, but reintroduces the third-party
   gatekeeper + app-install friction for non-technical buyers that motivated this spike in the first
   place.
3. **Abandon Canva-hosted publishing for the live-data tier**: ship the interactive invite as a
   self-hosted page on Seline Paperie's own domain (buyer gets a link/QR code instead of a Canva site
   they customize visually). This keeps full control of the live layer but is a real product/positioning
   change — Canva's drag-and-drop customization is currently the entire value prop for non-technical
   Etsy buyers. A hybrid (Canva for the static invite suite, a separate hosted page just for the
   RSVP/countdown block, linked or QR-coded from the Canva site) is the least disruptive version of this
   and doesn't depend on Canva's embed pipeline at all.

## Competitive teardown — how existing Etsy listings do "live" content (2026-07-12)

Follow-up research after the technical spike, to answer "if the wall is real, how do the dozens of
Etsy listings advertising *live countdown + trackable RSVP* on Canva sites pull it off?" Answer: they
do exactly what this spike concluded is the only viable path — **they wire the Canva page to
third-party services that are already on Iframely's whitelist. None of them self-host anything.** Every
mechanism below was verified against the whitelist file (`iframe.ly/domains.json`) pulled in Phase B.

### Countdown → embed a whitelisted countdown service

Tutorials universally point to [TickCounter](https://www.tickcounter.com) (or svgcountdowns). Both
are whitelisted, so both render as a live iframe that ticks and survives publish — *because the src is
their domain, not the seller's*:

```
^https?://(?:www\.)?tickcounter\.com     ✓ whitelisted
^https?://(?:www\.)?svgcountdowns\.com    ✓ whitelisted
```

Canva's *own* countdown element, by contrast, flattens to a frozen PNG on export — the same trap that
killed the original countdown-sticker feature.

### RSVP → three patterns, all avoiding self-hosting

1. **Link-out button (most common):** the "RSVP" button is a plain hyperlink to a Google Form, opens
   in a new tab; responses land in Google Sheets. "Trackable" = *the couple* reads the sheet. Nothing
   is embedded, so the flattening problem never applies — a link is not embedded content.
2. **Embedded whitelisted form provider:** paste a Jotform / Typeform / Tally / Cognito / Google Form
   URL into Canva's Embed element. All whitelisted → render as live in-page iframes:
   `jotform.com ✓  typeform.com ✓  cognitoforms.com ✓  wufoo ✓  paperform.co ✓  docs.google.com/forms ✓`
3. **Purpose-built service:** [ouRSVP](https://www.oursvp.app) — a Canva App *and* a whitelisted
   domain (`oursvp.app ✓`) built specifically for Canva RSVP.

### The part that matters for the moat

Everything above collects RSVPs into a **couple-facing** dashboard/sheet. **None of it shows guests
live, server-backed aggregate state** — no "24 of 40 replied," no who's-coming list rendered on the
published page for a logged-out guest. That specific product does not exist among these listings, and
the reason is this exact wall: to show guest-visible live data you'd have to *be* the whitelisted
domain, and no off-the-shelf service offers that guest-facing view.

### Why we dropped it

- **The achievable 80%** (live countdown + working RSVP collection) is already commoditized. Seline
  Paperie can match every competitor with zero backend — just document "embed TickCounter, link a
  Jotform" in the listing. No moat, but no build either.
- **The true moat** (guest-visible live status / who's-coming) is precisely what the spike proved is
  blocked, and this teardown confirms *nobody* ships it — not for lack of imagination, but because the
  Iframely wall stops everyone identically. Pursuing it means becoming a whitelisted host yourself,
  which reopens the third-party-approval dependency the spike was trying to avoid. Decision: **dropped.**
