# CANVA-CHECKLIST — manual Phase C

Run this by hand in Canva. This is the one thing the automated part of the spike couldn't test:
whether Canva's own Iframely integration behaves differently from the public, unauthenticated surface
probed in [FINDINGS.md](FINDINGS.md). Given the Phase B results, the expectation is that all four URLs
fail the same way (link-card/thumbnail at best, never a live iframe) — but this table is where that
expectation gets confirmed or overturned.

**Do not stop at step 1–2 (editor render).** H5 is the kill shot: Canva has already been seen to
flatten a live element to a static PNG at publish time (the countdown-sticker app). A pass in the
editor proves nothing — only the published page's DOM counts.

Test URLs (all under `https://kristengutierrez.github.io/seline-embed-spike/`):

| # | URL |
|---|---|
| 1 | `/plain/` |
| 2 | `/oembed/` |
| 3 | `/player/` |
| 4 | `/og/` |

For each URL, in order:

1. In a Canva **Website**-typed design, insert the URL via the native **Embed** element (not an app).
2. Does it render in the editor? Screenshot it either way.
3. **Publish** the site.
4. Open the published `*.my.canva.site` URL in an **incognito window, logged out, on a phone** (or
   phone-width responsive view — but a real phone is better, matches the guest experience).
5. Inspect the DOM at that spot. Is it an `<iframe src="kristengutierrez.github.io/...">`, or an
   `<img src=".../\_assets/media/...">`, or something else (a plain link card, empty space)?
6. If it's a live iframe: does the countdown/clock in `/widget` actually tick? Reload the published
   page 60 seconds later — has the displayed number changed?
7. Leave it 24h, then reload the published page again. Still live, or has it gone stale/broken
   (session-expiry style failure)?

## Results table

| # | URL | Renders in editor? | Published DOM element | `src` domain matches mine? | Ticks live (60s reload)? | Still live after 24h? | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `/plain/` | | | | | | |
| 2 | `/oembed/` | | | | | | |
| 3 | `/player/` | | | | | | |
| 4 | `/og/` | | | | | | |

## Reading the result

- **All four show `<img src=".../_assets/media/...">` or a plain link card** → confirms FINDINGS.md:
  Canva's Iframely integration uses the same (or an equally restrictive) domain whitelist as the public
  one. The verdict stands as "no, not without Iframely publisher approval."
- **Any of them shows a live `<iframe>` that ticks and survives 24h** → Canva's integration is more
  permissive than the public Iframely surface tested in Phase B. Worth re-reading FINDINGS.md's verdict
  — this would be a real reason to reopen the "buyer pastes URL into Embed, no app needed" path. Come
  back and tell me which hypothesis passed (H1/H2/H3) so the follow-up plan can target that exact
  mechanism.
- **Renders live in the editor but flattens to `<img>` on the published page** → this is H5, the exact
  failure mode that killed the original countdown-sticker feature. Confirms the editor is not a valid
  test surface for this question, full stop.
