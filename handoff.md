# Handoff: Sunday Order-Parsing Run

This file is instructions for the Claude running on this machine (her machine), written by Wes and his Claude. It describes one automation only: the weekly order-parsing run. It does **not** cover the email-drafting assistant for Etsy/website/Shopify inboxes — that's a separate, not-yet-designed piece (see "Not yet built" at the bottom). Don't start building that from this file.

## What this run is for

Every Sunday at noon, pull the week's orders from Shopify and run them through the parsing logic already built into `index.html` / `logic.js` in this repo (`starryKnightWebParser`), the same logic Wes uses when he runs orders by hand. Anything the deterministic logic can't figure out gets tagged exactly the way it already does today — never guessed silently. Then post a short status report as a GitHub issue.

## Steps, each run

1. **Pull orders.** Use the Shopify connector (`list-orders` / `get-order`) to pull orders placed since the last successful run. Don't ask Wes or her for a manual CSV export — that's the whole point of using the connector instead.

2. **Parse.** Run the orders through the existing parsing rules in `index.html`/`logic.js`. Also read `edge-cases.md` (in this same folder — create it if it doesn't exist yet) for patterns learned from past runs before falling back to a plain tag.

3. **Don't guess silently on ambiguous cases** (e.g. the existing "Could not determine big runner size automatically... please set manually" case). If the Notes column on an order makes an ambiguous case resolvable — something a human would clearly read the same way — you can resolve it, but:
   - Say so plainly in the run's report: which order, what you inferred, and the text you based it on.
   - Add it to `edge-cases.md` as a **candidate**, not settled fact, until a human confirms the pattern generalizes. Mark it clearly, e.g. `- [candidate, unreviewed] "runner note says X" → treat as Y (seen on order #1234, 2026-08-31)`.
   - If you're not confident, tag it exactly like the deterministic logic already does. A tag is never a failure — it's the safety net working as intended.

4. **Report.** Open a GitHub issue on `wescratty/starryKnightWebParser` tagging @wescratty, summarizing: orders parsed, orders tagged for manual review (and why), any candidate edge-cases added this run, and anything that looked wrong or worth a human's attention. If the run crashes or fails partway, open an issue immediately with whatever partial output and error info you have — don't wait, don't retry silently, don't lose the failure.

## Hard rules

- Never edit `index.html` or `logic.js` yourself. Those are the tested, deterministic floor — if you think a rule in there is wrong, say so in the report, don't patch it.
- Never mark an order fulfilled, shipped, or otherwise change anything in Shopify. Read-only.
- Treat every entry in `edge-cases.md` you didn't personally see a human promote as a hypothesis, not ground truth — re-state your confidence in the report rather than treating it as settled.
- If the Shopify connector or the GitHub reporting step itself fails, don't fail silently. Leave *some* trace — even a local file — if you can't reach GitHub at all.

## One-time setup (Wes/her, not the weekly run)

- [ ] Connect the Shopify connector on her Claude account, scoped read-only to orders if the connector allows it.
- [ ] Make sure GitHub access (gh CLI auth, or a GitHub connector) is set up for `wescratty/starryKnightWebParser` so the report step can actually open issues.
- [ ] Create an empty `edge-cases.md` next to `index.html` if one doesn't exist.
- [ ] Set a scheduled wake on the Mac (`pmset repeat wake ...`) a few minutes before noon Sunday, and make sure Claude Desktop is set to launch at login so it's actually running when the Routine fires.
- [ ] Create the local scheduled task (Routine) bound to this machine, cron `0 12 * * 0` (adjust for her timezone), prompt: "Run the weekly order-parsing job per handoff.md."

## Not yet built (don't start on these from this file)

- The email-drafting assistant for Etsy, her website inbox, and Shopify customer emails (drafts a reply in her voice, waits for her approval, notifies her phone). Still deciding which email provider her business inbox runs on, and Etsy has no direct connector — replies live in Etsy's own message center, which likely needs a different approach (browser automation) than the other two inboxes. A separate handoff will cover this once that's worked out.
