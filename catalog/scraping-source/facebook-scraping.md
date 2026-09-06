# Facebook (Groups & Marketplace) — scraping source

- **category**: scraping-source
- **provider**: Facebook / Meta
- **reusable**: partial — same reusable "one-browser/one-context/persisted-login-session, N-page worker pool" scraper harness as [yad2-scraping](yad2-scraping.md); the post-parsing logic (localized structured-field regexes for price/attributes, gate rules) itself is Facebook/app-specific.
- **docs**: https://www.facebook.com (no public API for group posts; scraped directly via an authenticated browser session)

## Overview
Facebook Groups content isn't reachable through Meta's public Graph API for third-party read access at this scale, so the practical approach is a Playwright session that reuses a real, manually-created login (exported as browser storage state) rather than any credential-based auth. A worker-pool of pages inside one shared context lets many groups/listings be scraped concurrently without re-authenticating per page.

## Playbook

### Prerequisites
- Playwright (`playwright` or `playwright-core`), with a locally cached Chromium binary.
- A one-time **manual** login in a real (non-headless) browser window — Facebook's login flow (2FA, device checks, captcha) is not meant to be scripted, and attempting to automate credentials risks account lockout. Export that session's `storageState` to a JSON file and treat it as the "session token" going forward.

### Setup steps
1. Launch a headed Playwright browser once, navigate to the target site, and log in manually as a human.
2. Call `context.storageState({ path: "auth-state.json" })` to persist cookies + localStorage for that logged-in session.
3. For every subsequent scrape, create a **new context from that saved state** (`browser.newContext({ storageState: "auth-state.json" })`) instead of logging in again — this is what makes headless, unattended runs possible.
4. Before trusting a scrape run's results, add a liveness check: load the site's home page inside the restored session and check for a login form in the DOM. A site can silently invalidate a session's cookies server-side well before their stated expiry, especially under sustained automated traffic — an unnoticed stale session produces a scrape that "succeeds" with zero real results instead of erroring loudly.
5. Re-export `storageState` (repeat step 1-2) whenever the liveness check fails.

### Core pattern
```ts
import { chromium, type Browser, type BrowserContext, type Page } from "playwright-core";

const STATE_PATH = "auth-state.json";

export class SessionExpiredError extends Error {}

async function verifySessionLive(page: Page, homeUrl: string): Promise<void> {
  await page.goto(homeUrl, { waitUntil: "domcontentloaded" });
  await page.waitForTimeout(2000);
  const loginFormCount = await page.locator('input[type="password"]').count();
  if (loginFormCount > 0) {
    throw new SessionExpiredError(
      `Session at ${STATE_PATH} looks logged out. Re-authenticate manually and re-export storageState.`
    );
  }
}

// ONE browser + ONE context (the saved login) + N pages, reused across a
// worker pool — avoids the cost/risk of opening a fresh authenticated
// context per unit of work.
export async function withAuthenticatedBrowser<T>(
  opts: { pages: number; homeUrl: string },
  fn: (s: { browser: Browser; ctx: BrowserContext; pages: Page[] }) => Promise<T>
): Promise<T> {
  const browser = await chromium.launch({ headless: true });
  try {
    const ctx = await browser.newContext({ storageState: STATE_PATH });
    const pages = await Promise.all(Array.from({ length: opts.pages }, () => ctx.newPage()));
    await verifySessionLive(pages[0], opts.homeUrl);
    return await fn({ browser, ctx, pages });
  } finally {
    await browser.close();
  }
}

// Trusted (OS-level) scroll — a lazy-loading feed frequently only responds
// to real input events, not script-driven scrollTo/scrollBy.
async function trustedScroll(page: Page, steps: number, deltaY: number, stepDelayMs: number) {
  for (let i = 0; i < steps; i++) {
    await page.mouse.wheel(0, deltaY);
    await page.waitForTimeout(stepDelayMs);
  }
}

// A tiny worker-pool: N pages pull from a shared queue of "groups"/targets so
// concurrency is bounded by page count, not by target count.
async function runPool<T, R>(items: T[], concurrency: number, worker: (item: T, slot: number) => Promise<R>): Promise<R[]> {
  const results: R[] = new Array(items.length);
  let next = 0;
  await Promise.all(
    Array.from({ length: concurrency }, async (_, slot) => {
      while (next < items.length) {
        const i = next++;
        results[i] = await worker(items[i], slot % concurrency);
      }
    })
  );
  return results;
}
```

Rate-limiting/anti-bot handling, generically: keep concurrency low (a handful of pages sharing one context, not dozens), add pacing waits between navigations, and never persist or log the raw scraped session file or its cookie values.

### Env vars
(none — session-cookie based, no API key; the session file itself is a local secret artifact and must never be committed or logged)

### Gotchas
- Never script the login form itself (credentials + 2FA) — it invites account flags/lockouts and is explicitly against most platforms' automation terms. Manual login + exported session state is both safer and more durable.
- A "successful" page load with a valid HTTP 200 can still be an empty/soft-logged-out shell — always positively check for a signed-in-only element (or absence of the login form) before treating scraped content as real.
- Distinguish a session-content-preview field (e.g. a feed's truncated post text) from the full content reachable only after visiting the item's own page — cheap pre-filters run on the preview should be conservative (return "unknown" rather than a wrong guess) since the authoritative check happens once the full text is available.
- Free-text parsing (price/quantity/date extraction from human-written posts) needs a negative lookaround/lookbehind guard against a large multi-digit number being mis-sliced by a smaller pattern (e.g. matching the tail digits of a much bigger comma-grouped number). Verify regexes against a real large-number example, not just the common case.
- When de-duplicating scraped items across multiple sources/sections, fingerprint on a normalized **prefix** of the content, not the full string — genuine reposts/duplicates are often byte-identical at the start but have unrelated trailing text appended (a quoted comment, a different call-to-action) that would break a full-string comparison.
- A platform's relative timestamps ("2 hours ago", "3 days ago") are frequently the *only* thing exposed for recent content — normalize them into an absolute time by subtracting from "now" at scrape time rather than leaving them unparsed, but keep in mind the result inherits the source's own rounding granularity.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. Consumed to scrape listing-style content from Groups and recognized Marketplace URLs via a persisted login session, with the same session also reused to build outbound deep links into the platform's own messaging UI.
