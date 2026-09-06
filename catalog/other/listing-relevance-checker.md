# Listing Relevance Checker

- **category**: other
- **provider**: internal (custom, browser-automation-based)
- **reusable**: partial — the concurrency-limited browser worker pool and dead/taken-phrase classifier pattern are portable to any scraped-listing repo; the Facebook-session/Yad2 bot-wall handling is source-specific.
- **docs**: https://playwright.dev/docs/api/class-browsercontext#browser-context-storage-state

## Overview
A scheduled/manually-run background job that revisits every non-irrelevant listing's source URL through a real headless (session-authenticated) browser to detect dead/taken/removed listings and mark them irrelevant, keeping a scraped dataset from accumulating stale posts. It classifies each revisited page by racing a "real content rendered" selector against known dead/taken/bot-wall phrase lists, and auto-retries bot-walled pages with a headed browser.

## Playbook

### Prerequisites
- Playwright (`playwright-core`) with a cached Chromium binary available locally (e.g. installed via the Playwright MCP browser or `npx playwright install chromium`)
- A saved, logged-in browser session state file (`storageState` JSON) for any source that requires auth to view listing pages
- A datastore holding listings with at least: a source URL, a "checked at" timestamp, and a relevance/status field

### Setup steps
1. Export a logged-in session once from an interactive browser: log in manually, then call `context.storageState({ path: ... })` and save the resulting JSON outside version control.
2. Write a small "find cached Chromium" helper that locates the browser binary installed by whichever Playwright toolchain is already on the machine (avoids requiring a separate `playwright install` in CI/cron contexts).
3. Build a session-liveness preflight: on browser launch, navigate to the source site's home page and check for a login form before doing any real work — an expired/invalidated session should hard-fail loudly, not silently produce all-"alive" (false) results.
4. Build the dead/taken/bot-wall phrase lists per source (see Core pattern) from real observed "this content isn't available" / "already rented" / CAPTCHA copy.
5. Build a generic order-preserving concurrency-limited worker pool that hands each queued item a dedicated browser Page (`slot`).
6. Write the per-listing check: navigate, race "real content selector" vs. "dead-phrase text" vs. a timeout, classify the result, return a verdict.
7. Wire up the write-back: mark dead/taken listings irrelevant (and clean up any associated stored files as part of that same write), stamp everything else with a "checked at" time in batches, and leave bot-walled items untouched for a headed re-run.
8. Schedule the script (cron/CI) with sane defaults for `--days` (re-check window), `--concurrency`, and a `--dry` flag for safe testing.

### Core pattern
```ts
// dead-phrase classifier — pure function, no browser dependency, easy to unit test
type PageVerdict = "dead" | "taken" | "blocked" | null;

const DEAD_PHRASES: readonly string[] = [
  "this content isn't available",
  "this page isn't available",
  /* ...localized variants of "removed" / "not found" */
];

const TAKEN_PHRASES: readonly string[] = [
  "already rented",
  "no longer available",
  /* ...localized variants of "taken" / "sold" / "expired" */
];

const BOT_WALL_RE = /captcha|verifying your browser|are you a human|access denied/i;

function classifyListingText(params: { hay: string; loginWall?: boolean }): { verdict: PageVerdict; reason: string } {
  const { hay, loginWall } = params;
  const dead = DEAD_PHRASES.find((p) => hay.includes(p));
  if (dead) return { verdict: "dead", reason: "removed: " + dead };
  if (loginWall) return { verdict: "dead", reason: "login/redirect, no post" };
  const taken = TAKEN_PHRASES.find((p) => hay.includes(p));
  if (taken) return { verdict: "taken", reason: "taken: " + taken };
  if (BOT_WALL_RE.test(hay)) return { verdict: "blocked", reason: "bot-wall (retry with a headed browser)" };
  return { verdict: null, reason: "" };
}

// generic order-preserving concurrency-limited pool; `slot` indexes a dedicated Page
async function runPool<T, R>(items: T[], n: number, worker: (item: T, slot: number) => Promise<R>): Promise<R[]> {
  const results: R[] = new Array(items.length);
  let next = 0;
  await Promise.all(
    Array.from({ length: Math.min(n, items.length) }, async (_, slot) => {
      while (true) {
        const i = next++;
        if (i >= items.length) break;
        results[i] = await worker(items[i], slot);
      }
    })
  );
  return results;
}

// per-listing check: race "real content" vs "dead/taken copy" vs timeout
async function checkOne(page: import("playwright-core").Page, url: string) {
  await page.goto(url, { waitUntil: "domcontentloaded", timeout: 15000 });
  await Promise.race([
    page.waitForSelector('[data-real-content-marker]', { timeout: 6000 }).catch(() => null),
    page.waitForFunction(
      (phrases: string[]) => phrases.some((p) => document.body.innerText.includes(p)),
      [...DEAD_PHRASES, ...TAKEN_PHRASES],
      { timeout: 6000 }
    ).catch(() => null),
    page.waitForTimeout(3500),
  ]);
  const hay = (await page.evaluate(() => document.title + "\n" + document.body.innerText)).slice(0, 6000);
  return classifyListingText({ hay });
}

// session-liveness preflight — fail loudly instead of silently returning all-clear
async function verifySessionLive(page: import("playwright-core").Page, homeUrl: string) {
  await page.goto(homeUrl, { waitUntil: "domcontentloaded" });
  await page.waitForTimeout(2000);
  const loginFormCount = await page.locator('input[name="email"], input[type="password"]').count();
  if (loginFormCount > 0) throw new Error("Saved session appears logged out/stale — re-export storageState.");
}
```

### Env vars
- none directly — auth is via a saved browser session state file on disk, not an env var (source-specific: some integrations may instead need an API key or bot-detection bypass credential)

### Gotchas
- A dead/removed page can still render whatever wrapper element you're using as the "real content" selector, just filled with unrelated fallback/"suggested for you" content instead of the actual listing — if you don't explicitly check for dead-phrases you'll silently scrape and misattribute that fallback content to every dead listing you hit (confirmed in production: hundreds of unrelated records ended up sharing the same handful of images from one fallback module).
- Bot-wall detection and dead/taken detection are different signals with different handling: a bot-walled page should be left untouched for a later headed-browser retry, not marked dead/irrelevant and not stamped as checked — treating it as either would create false negatives or a permanent skip.
- Sites using headless-browser fingerprinting (e.g. PerimeterX/Radware-style walls) will pass headless traffic through a CAPTCHA that a headed (non-headless) browser skips — auto-chaining one headed retry pass for whatever got blocked keeps normal runs fully headless while still covering the blocked subset.
- Bot-wall vendors evolve their challenge page over time (e.g. switching from a CAPTCHA-labeled page to a plain "verifying your browser..." loader with no historical phrase match) — even server-side `fetch()` calls (not just browser automation) can get walled, and a stale phrase/regex list will silently misclassify the wall as "no content found" instead of "blocked". Revisit the fingerprint list periodically.
- `window.scrollBy`/`scrollTo` via `page.evaluate` does NOT reliably trigger some sites' lazy-loading in headless Chromium — use `page.mouse.wheel` (a trusted, OS-level input event) instead when infinite-scroll content needs to load.
- If your write-back path marks something irrelevant via a `set(..., { merge: true })` rather than `update()`, a listing deleted mid-run can get silently resurrected as a near-empty "phantom" record. Either use `update()` (which fails on a missing doc) or add a runbook/check to detect and clean up phantom records.
- When marking a listing irrelevant also involves deleting associated stored files (images, attachments), delete the files FIRST and write the DB update second — if the write happens first and the file deletion fails or crashes, you're left with orphaned files with no record pointing at them (harmless), rather than a record claiming files exist that don't.
- Stamp every checked-but-still-alive listing with a "last checked" timestamp so subsequent runs skip recently-verified listings — do this in batched writes (e.g. groups of a few hundred) rather than one write per listing.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. That adopter re-checks each scraped listing's source URL
across two distinct source sites (one requiring an authenticated session, the other fronted by a
bot-detection wall), fans the checks out across a worker pool, and marks dead/taken listings
irrelevant while stamping everything else as checked. A known open bug there: the "checked"
stamp write uses a merge rather than a strict update, so a listing deleted mid-run can be silently
resurrected as a near-empty phantom record.
