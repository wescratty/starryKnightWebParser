# StarryKnight Order Parser — Morning Workflow (UX walkthrough)

Working through what the client's actual morning routine looks like with
the tool, step by step, with what's built vs. still a decision. Companion
to `starryKnightHandoff.md` (project decisions/history) -- this doc is
the UX spec, that one's the log.

## Decision this round: dropping the Shopify auto-pull (for now)

Wes's call: keep it simple. She already has a working manual habit (export
CSV from Shopify, save to a folder) — no reason to build and maintain a
token-holding proxy just to automate a step she doesn't mind doing.
**Phase 2 (Shopify Admin API auto-pull) is shelved, not scoped further
for now.** This also simplifies hosting a lot: with no backend piece, we
just need somewhere to serve one static HTML file. See "Hosting" at the
bottom.

The single-file, upload-a-CSV design already built for phase 1 doesn't
need to change for this — it was already the right shape.

## The walkthrough

1. **Coffee.** No notes.

2. **Opens the bookmarked link in Chrome.**

   Once hosted (see Hosting below), this is a normal bookmark to the
   Cloudflare-hosted URL, not a local file. That matters for the next
   question:

   > Does index.html store the date range on her desktop? Does clearing
   > history nuke it?

   Two different things are getting conflated here, worth separating:

   - **The app's stored settings** (last-processed timestamp, custom
     colors/ignore words) live in the browser's `localStorage` for that
     site. That's a different storage bucket than the page cache, but
     Chrome's "Clear browsing data" dialog *can* wipe it too if "Cookies
     and other site data" is checked (not just "Cached images and
     files").
   - **Getting her to see a new version after a bug fix** doesn't
     actually require clearing anything that broad. The real fix belongs
     on the hosting side: set `Cache-Control: no-cache` (or similar) on
     `index.html` so the browser always revalidates before showing a
     cached copy. Cloudflare Pages can be configured this way with a
     `_headers` file in the repo. Once that's in place, a normal reload
     — or at most a hard refresh (Ctrl+Shift+R), which bypasses the page
     cache *without* touching cookies/localStorage — is enough to pick up
     a fix. She'd never need "clear all history" as a troubleshooting
     step at all.
   - Belt-and-suspenders: the Last Processed Order Timestamp field is
     already visible and editable in Settings, not hidden state. If
     localStorage ever does get wiped (she clears everything anyway, new
     machine, whatever), she can just look at the last cut sheet's date
     range and type it back in. Worth keeping that field visible by
     default rather than tucked away, given this.

   **Action:** add a `_headers` file to the Cloudflare Pages deploy
   setting no-cache on `index.html`. Not done yet -- part of the hosting
   setup step.

3. **Opens Shopify.**

   > Can it suggest the date range? "Next up: since 2026-08-23 20:12:32"

   Yes -- straightforward, since we already store the last-processed
   timestamp. Proposed: a small banner on the upload screen, visible
   before any CSV is even loaded, reading something like "Next up: export
   orders from **{last processed}** through today." Not built yet.

   > Can we store a link to her Shopify orders page, with a
   > `<Get New Orders>` button?

   Yes, a plain link/button that opens her store's Shopify admin orders
   page in a new tab is easy and safe (just a URL, no credentials
   involved). One caveat: I haven't confirmed whether Shopify's export
   dialog itself can be deep-linked with the date range pre-filled via
   URL parameters -- that's worth checking before promising it, so the
   button should be scoped as "opens her orders page" for now, with the
   suggested date range shown as plain text next to it for her to type
   in, rather than assuming the dates land pre-filled. Not built yet.

4. **Downloads `order.csv` to her ACTIVE folder.** No change -- normal
   Shopify export + browser download, already her habit.

5. **Back to our tab.** No notes -- it's a bookmark, not a re-navigation
   through anything.

6. **Opens `order.csv` in the app.** Already built (upload button +
   drag-and-drop).

7. **Makes her cut sheet.**

   > Cut sheet saved to ARCHIVED? Maybe an ACTIVE_HTML folder?
   > Can we still move order.csv to ARCHIVED? Browsers restrict file
   > activity, I know.

   You're right that a plain browser download can't reach into her
   folder structure -- it lands wherever Chrome's download setting points
   (Downloads, or wherever she's set as default), and a page can't move
   or rename files on her disk through a normal `<a download>` link.

   There is a real, standards-based way around this on Chrome/Edge
   specifically: the **File System Access API**
   (`showDirectoryPicker()`/file handles with read-write). It lets a
   page ask for permission to a specific folder *once*, remember that
   permission across sessions ("Allow on every visit," available since
   Chrome 122), and then read/write/rename files there directly --
   which would let the app write the cut sheet straight into an
   `ACTIVE_HTML` (or `ARCHIVE_HTML`) folder *and* move the processed CSV
   from `ACTIVE` to `ARCHIVE`, matching the original desktop tool's
   folder behavior almost exactly.

   Caveats worth knowing before committing to this: it's Chrome/Edge
   desktop only -- Firefox and Safari don't implement it (Mozilla has
   said they consider it out of scope, not just "not yet"), and it
   doesn't work on mobile in any browser. Since she's on desktop Chrome,
   this isn't a practical limitation for her, but it does mean the app
   would depend on being used in a browser that supports it.

   **Recommendation:** treat this as a "phase 1.5" add-on, not part of
   today's ship -- it's a real scope increase (folder permission UX,
   handling a revoked/denied permission gracefully, one-time setup
   flow) and deserves its own pass rather than getting bolted on. Not
   built yet. Worth your go-ahead before I start it.

8. **Prints the cut sheet.**

   > The size-button status colors disappear when printing.

   Confirmed and **fixed** -- browsers suppress background colors on
   print by default (an ink-saving default), unless the page opts out
   with `print-color-adjust: exact`. Added that to the report's print
   CSS and verified in a headless print-preview render: the yellow/
   green/red/gray states now survive printing and PDF export. Shipped
   in the latest `index.html`.

## Revised phasing

1. ~~Phase 1: manual CSV upload~~ -- **done, shipped.**
2. ~~Phase 2: Shopify auto-pull~~ -- **shelved.** Keeping her existing
   manual export habit; not worth the token-proxy complexity for a step
   she doesn't mind doing.
3. **Phase 1.5 (candidate, not started):** the "Next up" date-range
   banner + Shopify link button (item 3 above), and the File System
   Access folder integration (item 7 above) -- both quality-of-life,
   neither required to use the tool. Proposing these as a follow-up
   round once hosting is settled, pending your go-ahead.

## Hosting (now much simpler without phase 2)

With no proxy/token-holding backend needed, hosting is just "serve one
static HTML file" -- no compute, no secrets to manage. That reopens the
GitHub Pages vs. Cloudflare Pages question from earlier in a simpler
light:

- **Cloudflare Pages** -- free static hosting, keeps the GitHub repo
  private if you want the source non-public, pairs natively with
  Cloudflare Access for a login wall if you decide to gate it, and
  reuses infrastructure you already run for the WTC site (open question:
  same Cloudflare account as WTC, or a separate one for client/brand
  separation -- still your call).
- **GitHub Pages** -- also free, but on the free plan requires the repo
  to be public to publish, and the published page itself is never
  access-gated on any plan (Enterprise-only feature). Simpler to wire up
  right now (repo's already on GitHub), but doesn't answer the
  access-gating question if that turns out to matter to you.

Both need the `_headers`/cache-control fix from item 2 above regardless
of which one you pick, so that's not a differentiator.
