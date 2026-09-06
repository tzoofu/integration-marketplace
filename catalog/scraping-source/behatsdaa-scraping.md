# Behatsdaa (scraping source)

- **category**: scraping-source
- **provider**: Behatsdaa (behatsdaa.org.il — Israeli employee-benefits/rewards portal)
- **reusable**: partial — Behatsdaa-specific parsing (Hebrew labels, order-history layout) and manual-login flow, but the "Playwright-driven browser automation packaged as an agent skill, with the user handling login" pattern is reusable for any scraping-source integration where the site can't be scripted end-to-end. See [yad2-scraping](yad2-scraping.md) for the sibling pattern (there, a spoofed-UA automated scraper; here, a session-based manual-login one).
- **docs**: https://www.behatsdaa.org.il (no public API; portal scraped directly by an authenticated human session)

## Overview
Behatsdaa is a benefits/rewards portal with no scriptable login (2FA/session flows that resist automation) and a page layout that's known to shift between visits. Rather than a fully-automated scraper, the practical pattern here is an **agent-driven, human-in-the-loop browser session**: the agent drives Playwright navigation/extraction, but a real person completes the login step interactively, and the agent re-confirms page structure (via a fresh snapshot) each run instead of hardcoding brittle selectors.

## Playbook

### Prerequisites
- Playwright browser automation available to the driving agent/script (e.g. an MCP-exposed `browser_navigate`/`browser_snapshot`/`browser_click` tool set).
- A human available to complete login interactively — no stored credentials, no scripted 2FA bypass.
- Read/write access to wherever scraped results get reconciled (a data store of previously-seen codes) so found items can be checked against what's already known before adding anything.

### Setup steps
1. Open the target site's login page in a real (headed, human-visible) browser session.
2. Prompt the human operator to complete login manually; wait for their explicit confirmation before proceeding — never attempt to detect "logged in" purely by page content, since an interstitial or partial-load page can look deceptively similar.
3. Navigate to the target data page (e.g. an order/reward history view). Because the page layout is known to move between visits, don't hardcode a URL path or DOM structure as gospel — re-derive it from a fresh accessibility snapshot each run, and ask the operator if the expected content isn't where you expect.
4. Apply any available status/category filter in the UI (e.g. "available" vs "redeemed/expired") *before* extracting rows, rather than pulling everything and filtering client-side — it avoids scraping and then discarding stale/irrelevant data.
5. For expandable detail rows (e.g. "order details" panels), click through and expand *all* of them first, then take one final full-page snapshot — cheaper and more reliable than re-snapshotting after every single click.
6. Cross-check every extracted item against what's already stored downstream (matching on a **normalized** identifier — see Gotchas) before deciding what's actually new.
7. Show the human a clear diff (new vs. already-known) and get explicit confirmation before writing anything.

### Core pattern
```markdown
<!-- Shape of an agent-skill driving this kind of "human logs in, agent
     extracts" scrape. Pseudocode narrating the actual tool calls an agent
     would make via a Playwright MCP surface. -->

1. browser_navigate(loginUrl)
   -> tell the human: "Browser is open at the login page — please log in,
      then tell me when you're done."
   -> WAIT for explicit human confirmation (do not poll/guess).

2. browser_navigate(targetPageUrl)
3. browser_snapshot()
   -> locate the status/category filter control from the snapshot; if one
      exists, browser_click it to the "available"/"active" state before
      reading rows.

4. From the snapshot, collect one record per row: {id, label, value, date,
   detailButtonRef}. Treat visually-identical repeated rows as separate
   instances, not duplicates to collapse — a user can legitimately hold two
   coupons/orders for the same item.

5. For each row: browser_click(detailButtonRef)   // expand, don't re-snapshot yet
   Then ONE browser_snapshot() after all rows are expanded, to read every
   detail panel's code/reference fields at once.

6. Normalize + compare against existing stored records (strip whitespace/
   punctuation from identifiers before comparing — see Gotchas).

7. Present a table: {new_or_existing, id, label, value} to the human.
   Only write new records after explicit confirmation.
```

Rate-limiting/anti-bot handling, generically: because this pattern relies on a real human session rather than a bot pretending to be one, there's no bot-wall to defeat — the main discipline is *pacing UI interactions realistically* (small waits between clicks/navigations) so the flow doesn't look like a scripted mass-click, and never attempting to script the login/2FA step itself.

### Env vars
(none — session-based manual login; no stored credentials or tokens)

### Gotchas
- Never try to script past a login/2FA flow that the site clearly doesn't want automated — hand that step to a human and only drive the browser afterward. This is both a reliability and a terms-of-service consideration.
- A page's layout/URL can genuinely change between scraping sessions on some sites — treat the previous run's exact selectors/paths as a hint, not a guarantee, and re-derive from a fresh snapshot rather than failing silently on a stale assumption.
- When cross-referencing a scraped identifier/code against existing records, **normalize before comparing** (strip separators like `-`, collapse whitespace) — the same underlying value can be formatted differently between visits or between the source site and your own storage, and a naive exact-string match produces false "this is new" positives.
- Don't invent a downstream field the source page doesn't actually show (e.g. don't back-fill an expiry date from a purchase date) — a missing field should stay missing, not get a plausible-looking guess.
- Repeated identical-looking rows (same item purchased more than once) are legitimate separate instances — don't de-dup by content alone.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. Consumed to pull redeemable reward/coupon codes from a user's order-history page and reconcile them against an existing local store of known codes, with a human completing login manually since the flow isn't scriptable.
